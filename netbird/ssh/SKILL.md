---
name: netbird-ssh
description: Configuring and troubleshooting Netbird's built-in SSH server, dashboard SSH access, and OAuth2 re-authentication on self-hosted deployments with embedded Dex. Use when enabling SSH in the Netbird dashboard, diagnosing "SSH connection failed" errors, removing the credentials dialogue, or understanding the per-window re-authentication requirement.
---
# Netbird SSH — Configuration & Troubleshooting

## Overview

Netbird v0.66+ includes a built-in SSH server on port **22022** on each peer. The Netbird dashboard uses this server for browser-based SSH. The standard system SSH daemon is unaffected.

## SSH Flags (v0.66.2)

SSH is configured via `netbird up` — **not** via the systemd service:

| Flag | Purpose |
|---|---|
| `--allow-server-ssh` | Enable the built-in SSH server on port 22022 |
| `--enable-ssh-root` | Allow root login (dashboard always connects as root) |
| `--disable-ssh-auth` | Skip JWT/OIDC credentials dialogue — any ACL-authorized peer can connect |
| `--enable-ssh-local-port-forwarding` | Allow SSH local port forwarding |
| `--enable-ssh-remote-port-forwarding` | Allow SSH remote port forwarding |
| `--enable-ssh-sftp` | Allow SFTP |
| `--ssh-jwt-cache-ttl int` | JWT cache TTL for SSH auth (seconds) |

**⚠️ `--enable-ssh` does NOT exist in v0.66.2.** Using it in a systemd drop-in or `netbird up` causes a crash-loop. Always use `--allow-server-ssh`.

## Recommended Home Lab Configuration

```bash
netbird up --allow-server-ssh --enable-ssh-root --disable-ssh-auth
```

- Enables dashboard SSH, allows root login, removes credentials dialogue
- Any ACL-authorized peer can connect without a password
- Do **not** use `--disable-ssh-auth` in production environments

## Applying Flags by Host Type

### Bare Metal / Direct Hosts (aspire, pve2, pve3, vps1)

```bash
netbird up --allow-server-ssh --enable-ssh-root --disable-ssh-auth
```

Applying new flags to an **already-connected** peer does **not** disconnect it — `netbird up` returns "Already connected" and applies the flags immediately.

### LXC Containers via pct exec

Containers require `netbird down && netbird up` to apply new flags:

```bash
# dns1 (CT107) and dns2 (CT108) on pve3
ssh pve3 "pct exec 107 -- sh -c 'netbird down && netbird up --allow-server-ssh --enable-ssh-root --disable-ssh-auth'"
ssh pve3 "pct exec 108 -- sh -c 'netbird down && netbird up --allow-server-ssh --enable-ssh-root --disable-ssh-auth'"

# nb1 (CT111) on pve2
ssh pve2 "pct exec 111 -- sh -c 'netbird down && netbird up --allow-server-ssh --enable-ssh-root --disable-ssh-auth'"
```

**⚠️ Broken drop-ins:** dns1/dns2 had systemd drop-ins with invalid `ExecStart` overrides that prevented `netbird up` from working. Remove any `ExecStart` lines from drop-ins before applying flags. Only `Environment=` lines are valid in Netbird drop-ins.

## API: Enabling SSH on a Peer

```bash
curl -X PUT https://netbird.bartschnet.de/api/peers/<peer-id> \
  -H "Authorization: Token <api-token>" \
  -H "Content-Type: application/json" \
  -d '{"ssh_enabled": true}'
```

**Note:** The API `ssh_enabled` field only enables/disables the SSH server. It does **not** control CLI flags like `--enable-ssh-root` or `--disable-ssh-auth`. Those must be set via `netbird up` on the peer itself.

## Dashboard SSH Mechanism

1. User opens Netbird dashboard → Peers → SSH button on a peer
2. Management server creates an **ephemeral WireGuard peer** in the browser
3. A "Temporary access policy for peer firefox-128-browser-client" is auto-created
4. Browser connects to `<peer-netbird-ip>:22022` (Netbird SSH server)
5. An ephemeral SSH key pair is used — no password if `--disable-ssh-auth`
6. Temporary peer and policy are cleaned up after the session

**Root login:** The dashboard always connects as `root`. Without `--enable-ssh-root`, the connection fails silently ("SSH connection failed. Check the console for details.").

### Common Dashboard SSH Failures

| Symptom | Cause | Fix |
|---|---|---|
| "SSH connection failed" on all peers | `--enable-ssh-root` not set | `netbird up --allow-server-ssh --enable-ssh-root --disable-ssh-auth` on each peer |
| "SSH connection failed" on one peer | Peer not running `--allow-server-ssh` | Same fix on that peer |
| Credentials dialogue on every new window | OAuth2 per-window re-auth (see below) | Use `--disable-ssh-auth` |
| Android peer has no SSH button | Android Netbird client does not support SSH server | Expected — no fix |
| Peer shows "Disconnected" in dashboard | Peer daemon not running | `sudo systemctl start netbird` |

## OAuth2 Re-authentication per SSH Window

Each new browser window opened from the Netbird dashboard SSH button triggers a full OIDC login flow. This is a known limitation of the embedded Dex IDP.

### Root Cause

`SSHCredentialsModal.tsx` opens SSH via `window.open()` — the new window inherits zero OIDC tokens.

The embedded Dex IDP (`management/server/idp/embedded.go`) has hardcoded limitations:
- No `offline_access` scope
- No `refresh_token` grant
- Local connector does NOT support `prompt=none` (silent token renewal)

`OIDCProvider.tsx` uses `service_worker_only: false` — tokens are stored in JS memory only, never in a service worker or shared storage. Silent renewal via `prompt=none` iframe returns `login_required` from Dex, triggering a full redirect.

Evidence in `idp.db`: `offline_session = 0`, `refresh_token = 0`, abandoned `prompt=none` auth_requests.

### Workarounds

| Option | Effect |
|---|---|
| `--disable-ssh-auth` on all peers | No credentials dialogue at all (already applied to all Bartschnet Linux hosts) |
| Replace embedded Dex with Authentik/Keycloak/Zitadel | External OIDC supports `offline_access` + `prompt=none` — one login per browser session |

## Systemd Drop-in: Only Environment Variables

Netbird's systemd service only accepts `Environment=` lines in drop-ins. SSH flags cannot be set here.

**Valid drop-in (`/etc/systemd/system/netbird.service.d/*.conf`):**
```ini
[Service]
Environment="NB_DISABLE_FIREWALL=true"
```

Adding `ExecStart` overrides or unknown flags to drop-ins causes crash-loops or silent failures.

## aspire: Outbound SSH Intercept for Netbird IPs

On aspire, `/etc/ssh/ssh_config.d/99-netbird.conf` intercepts outbound SSH to Netbird IPs (`100.64.0.0/10`) and routes them through `netbird ssh proxy` on port 22. This is the **outbound SSH client** intercept — separate from the Netbird SSH server (port 22022).
