---
name: unattended-upgrades
description: Configure and enable unattended-upgrades (automatic security/package updates) on Debian/Ubuntu hosts, including third-party repositories such as NetBird.
---

# Unattended-Upgrades Skill

## Overview

`unattended-upgrades` is a package that automatically installs security (and optionally other) updates on Debian/Ubuntu systems. This skill covers:

- Installing unattended-upgrades
- Enabling periodic runs via `20auto-upgrades`
- Adding third-party repositories to allowed origins using drop-in config files
- Verifying configuration with a dry run

---

## Installation

On Debian hosts where `unattended-upgrades` is not pre-installed:

```bash
apt-get install -y unattended-upgrades
```

Ubuntu 24.04 (noble) typically ships with it pre-installed.

---

## Configuration Approach

**Always use drop-in files** — never modify `/etc/apt/apt.conf.d/50unattended-upgrades` directly, as it is distribution-managed and may be overwritten by package upgrades.

Drop-in files go in `/etc/apt/apt.conf.d/` with a numeric prefix higher than 50, e.g. `51unattended-upgrades-<service>`.

---

## Adding a Third-Party Repository (e.g. NetBird)

### 1. Identify origin metadata

Inspect the repository's InRelease file to find the correct origin fields:

```bash
cat /var/lib/apt/lists/<encoded-repo-url>_dists_<suite>_InRelease | grep -E "^(Origin|Label|Suite|Codename):"
```

For NetBird (`pkgs.netbird.io`):
```
Origin: Artifactory
Label: Artifactory
Suite: stable
```

> **Important:** Do NOT use `origin=Artifactory` — it is too generic and would match any Artifactory-hosted repo. Use `site=pkgs.netbird.io` to target the specific host.

### 2. Create the drop-in file

```bash
cat > /etc/apt/apt.conf.d/51unattended-upgrades-netbird << 'EOF'
Unattended-Upgrade::Origins-Pattern {
    "site=pkgs.netbird.io";
};
EOF
```

This works on both Ubuntu (which uses `Allowed-Origins`) and Debian (which uses `Origins-Pattern`) — `Origins-Pattern` is the newer, more flexible format and coexists correctly with legacy `Allowed-Origins` blocks on Ubuntu.

---

## Enable Periodic Runs

Create `/etc/apt/apt.conf.d/20auto-upgrades` if not present:

```bash
cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
EOF
```

Then enable and start the apt timers:

```bash
systemctl enable --now apt-daily.timer apt-daily-upgrade.timer
```

Check timer status:
```bash
systemctl status apt-daily.timer apt-daily-upgrade.timer
```

---

## Verification

Run a dry-run to confirm allowed origins and which packages would be upgraded:

```bash
unattended-upgrade --dry-run -d 2>&1 | grep -i "allowed origins\|<package-name>"
```

Expected output for NetBird:
```
Allowed origins are: ..., site=pkgs.netbird.io
Checking: netbird (...)
Checking: netbird-ui (...)
netbird
netbird-ui
```

---

## Notes

- **Debian 13 (trixie)**: `unattended-upgrades` is not pre-installed. Default allowed origins are:
  - `origin=Debian,codename=trixie,label=Debian`
  - `origin=Debian,codename=trixie,label=Debian-Security`
  - `origin=Debian,codename=trixie-security,label=Debian-Security`
- **Ubuntu 24.04 (noble)**: Pre-installed with ESM origins already configured.
- **Remote hosts via SSH**: Use `ssh root@<ip>` — Netbird IPs are used for connectivity.
- **aspire (local)**: Requires `sudo` for writes to `/etc/apt/apt.conf.d/`.
