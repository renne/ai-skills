---
name: tcp-keepalive
description: Linux TCP keepalive tuning — configuring tcp_keepalive_time, tcp_keepalive_intvl, and tcp_keepalive_probes to keep long-lived TCP connections alive through NAT devices (routers, firewalls) and to detect dead connections faster. Use when VPN tunnels, gRPC streams, WebSocket connections, or SSH sessions drop after long idle periods, or when a router/firewall silently drops NAT entries for idle connections.
---
# Linux TCP Keepalive Tuning

## Overview

TCP keepalive probes are sent by the OS on idle TCP connections to verify the remote end is still alive and to keep NAT state table entries active in intermediate routers.

**Default values** (Linux):
```
tcp_keepalive_time   = 7200s  (2 hours — too slow for most NAT devices)
tcp_keepalive_intvl  = 75s
tcp_keepalive_probes = 9
```

Most home routers (e.g., FritzBox) drop idle TCP NAT entries after **60–90 minutes**. With the default 2-hour idle time before probing, connections silently die before the OS detects or prevents it.

---

## Recommended Values (Behind NAT / Home Router)

```bash
sudo tee /etc/sysctl.d/99-tcp-keepalive.conf << 'EOF'
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6
EOF
sudo sysctl -p /etc/sysctl.d/99-tcp-keepalive.conf
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `tcp_keepalive_time` | 60 | Send first probe after 60s of TCP-level idle |
| `tcp_keepalive_intvl` | 10 | Retry every 10s if no ACK |
| `tcp_keepalive_probes` | 6 | Declare dead after 6 missed probes (60s total) |

**Result**: The OS sends a keepalive every ~60s. Routers see traffic and keep NAT entries alive. Dead connections are detected within ~120s (60s idle + 6×10s probes).

---

## Verify

```bash
# Check current values
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes

# Apply without reboot
sudo sysctl -p /etc/sysctl.d/99-tcp-keepalive.conf
```

The `/etc/sysctl.d/` file persists across reboots automatically.

---

## When to Use

| Symptom | Cause | Fix |
|---------|-------|-----|
| VPN relay/WebSocket drops after ~60–90min | Home router NAT timeout (no TCP keepalive) | Lower `tcp_keepalive_time` to 60 |
| gRPC stream dies after uptime | Same: router drops idle TCP | Same fix |
| SSH session hangs after being idle | Remote server or NAT dropped connection | Same fix, also set `ServerAliveInterval 60` in `~/.ssh/config` |
| Long-lived HTTPS/WebSocket connection dies | Router NAT entry expired | Same fix |

---

## Scope

TCP keepalive is a **system-wide setting** that applies to all TCP connections on the host, including:
- VPN control plane (relay, gRPC)
- HTTP/WebSocket connections
- Database connections
- SSH sessions

Applications can also set per-socket keepalive via `SO_KEEPALIVE` + `TCP_KEEPIDLE`/`TCP_KEEPINTVL`/`TCP_KEEPCNT`, which overrides the system defaults for that socket.

---

## Notes

- TCP keepalive (OS level) is different from application-level keepalive (e.g., WireGuard `PersistentKeepalive` = UDP, or gRPC ping = application message). Both can be needed simultaneously for different protocol layers.
- The setting has no meaningful impact on normal connection performance.
- IPv6 uses the same sysctl parameters.
