---
name: connectivity-troubleshooting
description: Troubleshooting Netbird connectivity failures on Linux peers behind NAT, covering WireGuard ENOKEY errors, relay WebSocket TCP connection drops, FritzBox/router NAT timeout killing long-lived TCP connections, stale nftables state requiring reboot, and systemd cleanup overrides. Use when peers lose connectivity after uptime, when "netbird down" is needed to restore networking, when ENOKEY errors appear in logs, or when the network recovers after rebooting.
---
# Netbird Connectivity Troubleshooting (Linux behind NAT)

## Symptom Pattern

After some uptime (typically 45–120 minutes), the Netbird tunnel stops working:
- All traffic to VPN peers fails silently
- DNS to internal resolvers (e.g., `10.0.0.7:53`) fails with `ENOKEY`
- Internet may also fail if DNS hijacking is present (see `dns/SKILL.md`)
- `netbird down && netbird up` or reboot restores connectivity
- Netbird log shows: `write: required key not available`

---

## Root Cause: WireGuard ENOKEY

`ENOKEY` = kernel error — WireGuard has no valid session key for a peer.

**WireGuard session key lifecycle**:
- Session keys expire after **~3 minutes** of active data or **~9 minutes** total
- To renew, a **WireGuard handshake** must succeed
- Handshake requires reachability via direct UDP or relay **AND** the Signal server

If the relay or signal server TCP connection drops and reconnect takes >3 minutes → existing keys expire → ENOKEY on any packet.

---

## Root Cause: FritzBox/Router TCP NAT Timeout

FritzBox (and many home routers) silently drop **long-lived TCP connections** from their NAT state table after ~60–90 minutes of inactivity at the TCP keepalive level.

**Netbird relay architecture**:
```
aspire → FritzBox → Internet → vps1:443 → Traefik → netbird-relay-server
```

The relay connection is a **WebSocket over HTTPS** (persistent TCP). If no TCP keepalive probes are sent, the OS default (`tcp_keepalive_time=7200s` on Linux) means no probes for 2 hours → FritzBox silently drops the NAT entry → relay WebSocket dies.

**Evidence in relay server logs** (on `vps1`):
```
healthcheck timeout for peer: sha-<hash>=   ← relay sees peer gone
closing peer connection                       ← relay tears down connection
```

**Timeline pattern**:
1. Relay connects at T+0
2. FritzBox drops NAT entry at ~T+73min (no TCP keepalive)
3. Relay healthcheck timeout logged on server
4. Netbird client starts reconnecting (exponential backoff)
5. Reconnect gap >3min → WireGuard session keys expire → ENOKEY
6. Even after relay reconnects, WireGuard needs a new handshake

**Note**: WireGuard's `PersistentKeepalive` (UDP, 25s, data plane) is completely separate from the relay TCP keepalive (control plane). Keepalive on one does NOT keep the other alive.

---

## Fix: TCP Keepalive Tuning

Lower the OS TCP keepalive timer so the relay TCP connection stays alive through FritzBox's NAT timeout:

```bash
sudo tee /etc/sysctl.d/99-tcp-keepalive.conf << 'EOF'
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
EOF
sudo sysctl -p /etc/sysctl.d/99-tcp-keepalive.conf
```

**Effect**: OS sends the first TCP keepalive probe after 60s of idle, then every 10s for up to 6 probes (total: 120s to detect dead connection). FritzBox sees the keepalive ACKs and keeps its NAT entry alive.

**Verify**:
```bash
sysctl net.ipv4.tcp_keepalive_time   # should return 60
```

The file persists across reboots. This is a system-wide setting that applies to all TCP connections including the relay WebSocket.

---

## Fix: Netbird Cleanup Service Override

Sometimes `netbird down` fails to restore connectivity because stale nftables rules or the `wt0` WireGuard interface remain. This requires a reboot. Fix with a systemd override:

```bash
sudo mkdir -p /etc/systemd/system/netbird.service.d/
sudo tee /etc/systemd/system/netbird.service.d/cleanup.conf << 'EOF'
[Service]
ExecStopPost=-/sbin/ip link delete wt0
ExecStopPost=-/usr/sbin/nft delete table netbird
EOF
sudo systemctl daemon-reload
```

The `-` prefix means failure is ignored (in case the interface/table doesn't exist). This runs on every `netbird stop` / `netbird down`, ensuring a clean state.

---

## Diagnostics

### Check ENOKEY pattern
```bash
# How often does ENOKEY occur and on which days?
sudo grep "required key not available" /var/log/netbird/client.log \
  | awk -F'T' '{print $1}' | sort | uniq -c

# What time of day does the relay disconnect?
sudo grep -E "healthcheck|relay.*timeout|connection.*closed" /var/log/netbird/client.log \
  | tail -30
```

### Check relay stability on vps1
```bash
# Replace with the peer's relay hashed ID
ssh vps1 "docker logs netbird-server 2>&1 | grep 'sha-<HASH>' | tail -20"
```

Peer's relay hashed ID can be found in the Netbird client log:
```
INFO shared/relay/client/client.go:194: create new relay connection: ..., local peer hashedID: sha-<HASH>=
```

### Check current TCP keepalive
```bash
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes
```

### Watch relay reconnections in real-time
```bash
sudo tail -f /var/log/netbird/client.log | grep -iE "relay|signal|reconnect|error|warn"
```

### Check if nftables table is stale after netbird down
```bash
sudo nft list tables  # should NOT show 'netbird' table after netbird down
ip link show wt0      # should NOT exist after netbird down
```

---

## Signal vs Relay vs WireGuard

| Layer | Protocol | Purpose | Keepalive |
|-------|----------|---------|-----------|
| WireGuard data plane | UDP | Encrypted traffic between peers | 25s PersistentKeepalive (keeps UDP NAT alive) |
| Relay (WebSocket) | TCP/TLS port 443 | Fallback tunnel when direct UDP fails | OS TCP keepalive (must be tuned) |
| Signal (gRPC) | TCP/TLS port 443 | Coordinates peer handshakes | Server-side 30s ping (hardcoded, not configurable) |
| Management (gRPC) | TCP/TLS port 443 | Delivers network config updates | Same as signal |

All three TCP connections go through the same FritzBox NAT. TCP keepalive tuning covers all of them.

---

## Netbird Log Location

Netbird does NOT log to journald by default. Logs are at:
```
/var/log/netbird/client.log
```
Reading requires `sudo`.

---

## References

- [Netbird Troubleshooting](https://docs.netbird.io/troubleshooting/netbird-client)
- WireGuard session key lifetime: `REKEY_AFTER_TIME` (180s), `REJECT_AFTER_TIME` (~9min)
