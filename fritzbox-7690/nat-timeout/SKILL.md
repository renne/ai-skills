---
name: nat-timeout
description: FritzBox 7690 NAT timeout behavior — how the FritzBox silently drops long-lived TCP connections from its NAT state table, causing VPN tunnels (WireGuard relay, gRPC, WebSocket) to fail after 60–90 minutes of no TCP keepalive activity. Use when investigating intermittent VPN disconnects, long-lived connection drops, or when connections fail after uptime but work fine after restart.
---
# FritzBox NAT Timeout for Long-Lived TCP Connections

## Problem

The FritzBox 7690 (and many home routers) maintains a NAT state table for all outbound connections. For TCP connections, the entry is kept alive as long as the connection is active. However, if no TCP keepalive probes are sent at the OS level, the FritzBox will silently drop the NAT entry after approximately **60–90 minutes** of no TCP-level activity.

This is distinct from TCP connection teardown — the OS still thinks the TCP connection is alive, but the FritzBox has removed its NAT mapping. New packets from the remote end can no longer be routed back to the client.

---

## Affected Connection Types

Any long-lived outbound TCP connection is affected:
- **VPN relay connections** (WebSocket over HTTPS, e.g., Netbird relay)
- **gRPC streams** (Netbird signal/management, QUIC fallback)
- **SSH connections** with no activity
- **HTTPS keep-alive connections** with long idle periods

---

## Symptom Pattern

1. Connection established successfully
2. Works fine for ~60–90 minutes
3. Silently fails — no error at OS level immediately
4. Remote end times out its healthcheck and closes connection
5. OS reconnect attempt may fail (FritzBox NAT entry already gone)
6. After reconnect succeeds, a new NAT entry is created and connection works again

**For Netbird specifically**: the relay WebSocket dies → relay healthcheck timeout logged server-side → Netbird reconnects → if reconnect gap >3 minutes, WireGuard session keys expire → `ENOKEY` errors on all tunnel traffic.

---

## Root Cause Analysis

Linux default TCP keepalive:
```
tcp_keepalive_time   = 7200s  (2 hours before first probe)
tcp_keepalive_intvl  = 75s
tcp_keepalive_probes = 9
```

With `tcp_keepalive_time=7200s`, the OS sends **no keepalive probes for 2 hours**. The FritzBox sees no activity on the TCP stream and removes its NAT entry at ~60–90min. When a packet finally arrives, the FritzBox has no mapping and drops it.

---

## Fix: Lower TCP Keepalive Timer

```bash
sudo tee /etc/sysctl.d/99-tcp-keepalive.conf << 'EOF'
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
EOF
sudo sysctl -p /etc/sysctl.d/99-tcp-keepalive.conf
```

**Explanation**:
- `tcp_keepalive_time = 60` — send first keepalive probe after 60s of idle
- `tcp_keepalive_intvl = 10` — retry every 10s if no ACK
- `tcp_keepalive_probes = 6` — give up after 6 missed probes (60s to declare dead)

FritzBox sees the keepalive ACKs every ~60s and keeps its NAT entry alive indefinitely.

**Verify** (survives reboot via `/etc/sysctl.d/`):
```bash
sysctl net.ipv4.tcp_keepalive_time   # → 60
```

---

## FritzBox Connection Behavior After Timeout

Once the FritzBox drops a NAT entry and the remote reconnects rapidly, the FritzBox may enter a confused state for multiple reconnect cycles:
- First connection: lasts ~73min (FritzBox NAT timeout)
- Rapid reconnections after first drop: last only 45s–2min15s each
- After the reconnect storm settles: new connection lasts hours again

This cascading behavior makes the issue look like repeated failures but is actually all caused by the single root cause: the initial NAT entry drop.

---

## Notes

- This is a **pure OS-level fix** — no FritzBox configuration change needed
- The fix applies to all TCP connections system-wide
- Application-level keepalive (e.g., Netbird's gRPC 30s ping) is separate and also helps, but TCP keepalive at the OS level is the authoritative mechanism
- The fix is permanent across reboots via `/etc/sysctl.d/`
