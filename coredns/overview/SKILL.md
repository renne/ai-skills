---
name: overview
description: CoreDNS overview covering what CoreDNS is, how to install it, Corefile syntax, server blocks, zones, plugin ordering, snippets, environment variables, and general operation. Use when setting up CoreDNS for the first time, writing or debugging a Corefile, understanding how server blocks and plugin chains work, or looking for general CoreDNS configuration guidance.
---
# CoreDNS Overview & Configuration

Source: https://coredns.io/manual/toc/

## What is CoreDNS?

CoreDNS is a fast, flexible DNS server written in Go. It chains **plugins** to form a DNS processing pipeline, making it highly composable. Each server block in the configuration file (`Corefile`) defines a set of zones and attaches one or more plugins to handle queries for those zones.

CoreDNS is the default in-cluster DNS server for Kubernetes but can also be used as a standalone resolver, authoritative server, forwarder, or any combination thereof.

## Installation

### Pre-compiled binaries

Download a release binary from https://github.com/coredns/coredns/releases, then run it directly:

```bash
./coredns -conf Corefile
```

### Docker

```bash
docker run --rm -p 53:53/udp -p 53:53/tcp \
  -v $(pwd)/Corefile:/etc/coredns/Corefile \
  coredns/coredns -conf /etc/coredns/Corefile
```

### From source (requires Go ≥ 1.21)

```bash
git clone https://github.com/coredns/coredns
cd coredns
make
```

### Startup flags

| Flag | Description |
|------|-------------|
| `-conf <file>` | Path to the Corefile (default: `./Corefile`) |
| `-dns.port <port>` | Default DNS port (default: `53`) |
| `-pidfile <file>` | Write PID to file |
| `-version` | Print version and exit |
| `-quiet` | Suppress version output |

## The Corefile

All configuration lives in a file named `Corefile`. On startup CoreDNS reads the Corefile in the current directory unless `-conf` overrides it.

### Server block syntax

```
[SCHEME://]ZONE[,ZONE...][:PORT] {
    PLUGIN [ARGUMENTS...]
    PLUGIN {
        option value
    }
}
```

- **SCHEME** – optional; defaults to `dns://`. Other values: `tls://`, `https://`, `grpc://`.
- **ZONE** – the DNS zone this block is authoritative/responsible for. Use `.` for the root (match everything). Comma-separate multiple zones.
- **PORT** – optional; defaults to `53`.
- **PLUGIN** – plugin name, optionally followed by arguments or a block.

### Minimal working example

```
.:53 {
    forward . 8.8.8.8 8.8.4.4
    log
    errors
}
```

This forwards all queries to Google DNS and logs queries and errors.

## Plugin Execution Order

The order of plugins **inside** a server block does **not** control execution order. Execution order is fixed at build time in `plugin.cfg`. This means you can list plugins in any order in your Corefile; CoreDNS will always run them in the correct sequence. Consult https://coredns.io/plugins/ for the authoritative sequence.

## Multiple Server Blocks

CoreDNS can serve multiple independent zones simultaneously:

```
example.org {
    file /zones/example.org.db
}

10.0.0.0/24 {
    file /zones/db.10.0.0.0
}

. {
    forward . 1.1.1.1
    cache
}
```

Each block is an independent server. A query is dispatched to the most-specific matching block.

## Multiple Zones in One Block

```
example.org, example.com {
    file /zones/example.org.db
    file /zones/example.com.db
}
```

## Listening on Specific Interfaces

Use the `bind` plugin to restrict which interfaces CoreDNS listens on:

```
.:53 {
    bind lo eth0
    forward . 8.8.8.8
}
```

## Comments

Lines starting with `#` are comments:

```
# This is a comment
.:53 {
    # forward all queries
    forward . 8.8.8.8
}
```

## Environment Variable Substitution

Reference environment variables in the Corefile using `{$VAR}` or `{%VAR%}`:

```
.:53 {
    forward . {$UPSTREAM_DNS}
}
```

Set the variable before starting CoreDNS:

```bash
export UPSTREAM_DNS=9.9.9.9
./coredns
```

## Snippets (Reusable Config Blocks)

Define a named snippet and import it in multiple server blocks to avoid duplication:

```
(common) {
    log
    errors
    prometheus :9153
}

example.org {
    file /zones/example.org.db
    import common
}

. {
    forward . 8.8.8.8
    cache
    import common
}
```

## DNS Transport Schemes

| Scheme | Protocol | Default Port |
|--------|----------|--------------|
| `dns://` | DNS over UDP/TCP (plain) | 53 |
| `tls://` | DNS over TLS (DoT) | 853 |
| `https://` | DNS over HTTPS (DoH) | 443 |
| `grpc://` | DNS over gRPC | 443 |

Example – listen on DoT:

```
tls://.:853 {
    tls /certs/server.crt /certs/server.key
    forward . 1.1.1.1
}
```

## Zone Delegation

CoreDNS respects DNS delegation via NS records in zone files. The `file` plugin can serve a parent zone while queries for delegated child zones fall through to another server block or upstream.

## Health & Readiness (Summary)

```
.:53 {
    health :8080        # /health endpoint for liveness probes
    ready :8181         # /ready endpoint for readiness probes
    forward . 8.8.8.8
}
```

Both return `200 OK` when the server is healthy/ready.

## Logging and Observability (Summary)

```
.:53 {
    log                 # query log to stdout
    errors              # error log to stdout
    prometheus :9153    # Prometheus metrics endpoint
    forward . 8.8.8.8
}
```

## Running CoreDNS as a systemd Service (bare-metal / LXC)

When not using Docker, install the binary and create a systemd unit:

```bash
# Download and install binary
VER=1.14.2
curl -fsSL "https://github.com/coredns/coredns/releases/download/v${VER}/coredns_${VER}_linux_amd64.tgz" | tar -xz
install -m 755 coredns /usr/local/bin/coredns
```

```ini
# /etc/systemd/system/coredns.service
[Unit]
Description=CoreDNS DNS Server
After=network.target

[Service]
ExecStart=/usr/local/bin/coredns -conf /etc/coredns/Corefile
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now coredns
```

---

## High-Availability CoreDNS Pattern (Two LXC Containers, Shared Corefile)

Run two identical CoreDNS containers for HA. Share a single Corefile via a **Proxmox bind mount** so both reload together:

1. **On the Proxmox host**, create the shared config directory:
   ```bash
   mkdir -p /opt/coredns
   # Edit /opt/coredns/Corefile here — single source of truth
   ```

2. **In `/etc/pve/lxc/<ctid>.conf`** for each container:
   ```
   mp0: /opt/coredns,mp=/etc/coredns,ro=1
   lxc.cgroup2.devices.allow: c 10:200 rwm
   lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
   ```

3. **In the Corefile**, include `reload 2s`:
   ```
   (common) {
       reload 2s
       ...
   }
   ```
   Both containers watch via inotify and reload within 2 seconds of any edit on the host — no restart needed.

4. **DNS servers configured on upstream routers**: primary `10.0.0.X`, secondary `10.0.0.Y` (the two container LAN IPs).

**Port conflict with NetBird**: If the container also runs a NetBird agent, its DNS proxy would conflict on port 53. Prevent this:
```bash
netbird up --disable-dns
```
Also set the "Routing Peers" group to **DNS Unmanaged Mode** in the NetBird dashboard to prevent the management server from pushing DNS config to the containers.

---

## Suffix Rewrite Pattern (apex domain bridging)

Use `rewrite stop name suffix` to dynamically mirror all hostnames under one apex to another. This is essential when the router's built-in DNS uses a different apex (e.g. `fritz.box`) than your preferred domain (e.g. `example.com`):

```corefile
example.com {
    # Rewrite *.example.com → *.fritz.box, rewrite answer back
    rewrite stop name suffix "example.com" "fritz.box" answer auto

    # Exceptions: names that exist in public DNS — stop before rewriting
    rewrite stop name exact vps1.example.com vps1.example.com

    # Forward rewritten (fritz.box) queries to the local router
    forward fritz.box 10.0.0.1

    # Forward public example.com names to authoritative DNS
    import deSEC

    cache 30
    log
    errors
}
```

- `answer auto` rewrites the answer section back (e.g. `docker.fritz.box` → `docker.example.com`).
- Put `rewrite stop name exact` rules for public names **before** the suffix rewrite so they are not rewritten.
- Dynamic DHCP hostnames (e.g. laptop, phone) are automatically mirrored as long as the router registers them — no manual record maintenance needed.

---

## References

- [CoreDNS Manual](https://coredns.io/manual/toc/)
- [CoreDNS GitHub](https://github.com/coredns/coredns)
- [Corefile man page](https://github.com/coredns/coredns/blob/master/corefile.5.md)
- [Plugin ordering (plugin.cfg)](https://github.com/coredns/coredns/blob/master/plugin.cfg)
