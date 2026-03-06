---
name: routing-troubleshooting
description: Troubleshooting NetBird routing issues on Linux peers, especially asymmetric routing caused by NetBird's policy routing table 7120, stale routes from removed peers, and static route overrides. Use when a host is reachable via LAN but not from a remote subnet routed through a gateway, when ping succeeds in one direction but DNS/TCP fails, or when NetBird's routing table contains unexpected subnets.
---
# NetBird Routing Troubleshooting

## How NetBird Manages Routing on Linux

NetBird installs **policy routing rules** and a **separate routing table (7120)** on every Linux peer. This is visible via:

```bash
ip rule list
# Output includes:
#   105: from all lookup main suppress_prefixlength 0
#   110: not from all fwmark 0x1bd00 lookup 7120

ip route show table 7120
# Shows subnets routed via the NetBird overlay (wt0)
```

**Rule priority order:**
1. `105`: look up `main` table — but **suppress default routes** (prefix length ≤ 0, i.e. `/0`)
2. `110`: look up `table 7120` — NetBird's overlay routes
3. `32766`: look up `main` — full main table including default

This means: for any destination with a route in table 7120, that route takes precedence over the main table's default gateway. A `/24` route in the main table **does** win (priority 105 finds it before reaching 110), but a route only in the default (`0.0.0.0/0`) does not.

---

## Symptom: Host Reachable From LAN but Not From Remote Subnet

**Scenario:** A CoreDNS/NetBird routing peer at `10.0.0.7` is reachable from `10.0.0.29` (same LAN), but not from `192.168.178.x` (routed via LAN-LAN WireGuard through the gateway).

**Root cause:** Table 7120 contains a route for `192.168.178.0/24 dev wt0`. Return packets from `10.0.0.7` to `192.168.178.x` are sent via the NetBird overlay (`wt0`) instead of the default gateway (`eth0`). Since the remote-subnet peer may be offline, traffic is blackholed.

**Diagnosis:**
```bash
# Check what's in table 7120
ip route show table 7120
# Look for unexpected subnets like 192.168.178.0/24

# Check where the routing decision lands
ip route get 192.168.178.1
# Bad: "192.168.178.1 dev wt0 ..."
# Good: "192.168.178.1 via 10.0.0.1 dev eth0 ..."
```

**Fix — add a static override in the main table:**
```bash
ip route add 192.168.178.0/24 via <gateway> dev eth0
```

Priority 105 (`lookup main suppress_prefixlength 0`) finds the `/24` in the main table before priority 110 checks table 7120. This overrides the wt0 route.

**Persist across reboots** (Debian/Ubuntu with ifupdown):
```
# /etc/network/interfaces — under the interface stanza
  post-up ip route add 192.168.178.0/24 via 10.0.0.1 dev eth0
```

---

## Stale Routes in Table 7120

NetBird peers can advertise subnet routes to other peers. When a peer is removed from the NetBird management server, the routes it advertised may **persist in table 7120** of connected peers even after the advertising peer is gone.

**Check what routes are active from the peer's perspective:**
```bash
netbird routes list
# Shows: Available Networks with network ID, subnet, and status (Selected/Excluded)
```

**Check via NetBird status:**
```bash
netbird status --detail
# Under each peer entry: "Networks: X.X.X.X/Y"
```

**Admin API check:**
```bash
curl -s -H "Authorization: Token $TOKEN" https://<mgmt>/api/routes | python3 -m json.tool
```

Note: Routes configured by individual peers (not via the admin API) may not appear in the routes API response, but will still be distributed via the network map. Use `netbird routes list` on the peer itself for the authoritative view.

**Resolving stale routes:**
1. **Remove the peer** from the NetBird management dashboard. Routes it advertised are withdrawn from all peers on next network map update.
2. **Restart NetBird** on the affected peer: `systemctl restart netbird` — re-fetches the network map.
3. **Add a static override** (see above) if the peer cannot be removed immediately.

---

## NetBird Routing Table Entries Explained

| Route in table 7120 | Meaning |
|---------------------|---------|
| `10.0.0.0/24 dev wt0` | This peer is a subnet router for `10.0.0.0/24` (advertised by self or another peer) |
| `192.168.178.0/24 dev wt0` | Another peer is advertising `192.168.178.0/24` as a routable subnet |
| `203.0.113.1/32 dev wt0` | A specific host route (e.g. a VPS) exposed via a subnet router peer |
| `100.64.0.0/10 dev wt0` | NetBird's own overlay range — always present |

---

## Verifying DNS Resolution Chain

When a DNS query fails on a host in a remote subnet (e.g. `aspire` at `192.168.178.x` querying `10.0.0.7:53`):

1. **Check if the nameserver IP is routable** from the querying host:
   ```bash
   ping 10.0.0.7  # Must succeed before DNS can work
   ```

2. **Check if the DNS server can send return packets correctly:**
   ```bash
   # On the CoreDNS host:
   ip route get 192.168.178.x
   # Must show eth0, not wt0
   ```

3. **Check systemd-resolved upstream** (on the querying host):
   ```bash
   resolvectl status
   # Shows which DNS server systemd-resolved is using per interface
   ```

4. **Test the DNS server directly** (bypasses systemd-resolved):
   ```bash
   dig @10.0.0.7 example.com
   ```

---

## Removing NetBird from a Linux Host (apt-installed)

```bash
apt-get purge -y netbird
apt-get autoremove -y
rm -f /etc/apt/sources.list.d/netbird.list
apt-get update -qq
```

The `purge` step runs NetBird's pre-uninstall script which stops the service and removes the WireGuard interface (`wt0`) cleanly. Verify:
```bash
which netbird          # should return nothing
ip link show | grep wt # should return nothing
```

Also remove the peer from the NetBird management dashboard to prevent stale route advertisements to other peers:
```bash
curl -X DELETE -H "Authorization: Token $TOKEN" https://<mgmt>/api/peers/<peer-id>
```
