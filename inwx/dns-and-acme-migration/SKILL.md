---
name: dns-and-acme-migration
description: Migrate domains from deSEC or another DNS provider back to INWX using the DomRobot XML-RPC API, including DNSSEC recovery and ACME DNS-01 reconfiguration for Proxmox or Traefik. Use when a zone is hosted elsewhere, nameserver delegation must move to INWX, or DNS-01 automation needs to switch to INWX safely.
---
# INWX DNS and ACME Migration

Use this skill when domains should be moved back to INWX as authoritative DNS while keeping service DNS, DNSSEC, and ACME DNS-01 working.

## What to verify first

1. Export the current authoritative zone from the active provider before changing delegation.
2. Confirm the current nameserver set and DNSSEC mode at INWX.
3. Inspect every ACME client that depends on DNS-01 so the provider switch can be done after the zone is present at INWX.

## DomRobot API notes

The INWX XML-RPC endpoint is:

```text
https://api.domrobot.com/xmlrpc/
```

Useful methods discovered in practice:

- `account.login`
- `account.unlock` for TOTP-based 2FA
- `domain.info`
- `domain.update`
- `nameserver.info`
- `nameserver.createRecord`
- `nameserver.deleteRecord`
- `nameserver.updateRecord`
- `nameserver.importZone`
- `dnssec.info`
- `dnssec.deleteall`
- `dnssec.enablednssec`

Important behavior:

- `domain.update` must use the parameter `ns`, not `nserver`
- `domain.info` returns the active nameserver set in `resData.ns`
- `nameserver.importZone` can be incomplete or lossy for some zones; verify record parity afterwards
- XML-RPC sessions depend on cookies. If you automate `account.login` and `account.unlock` yourself, use a cookie-aware transport or HTTP client; otherwise login and unlock may appear to succeed while later commands fail because the session was not preserved.

## Safe migration order

1. Export the live rrsets from the current provider.
2. Import or recreate the zone at INWX.
3. Compare the INWX zone against the source zone and correct any drift before switching delegation.
4. If the domain still has manual DNSSEC/DS state from the old provider, remove it with:

```text
dnssec.deleteall {domainName: <domain>}
```

5. Change registrar delegation with:

```text
domain.update {domain: <domain>, ns: ["ns.inwx.de", "ns2.inwx.de", "ns3.inwx.eu"]}
```

6. Re-enable DNSSEC at INWX with:

```text
dnssec.enablednssec {domainName: <domain>}
```

7. Verify `dnssec.info` shows `dnssecStatus: AUTO`.

DNSSEC changes can be asynchronous. After `dnssec.enablednssec`, INWX may report `AUTO` before authoritative nameservers publish the new `DNSKEY`, and the parent DS can lag or temporarily disappear while the chain stabilizes. If ACME DNS-01 fails with DNSSEC validation errors during that window:

1. Query authoritative `DNSKEY` directly from `ns.inwx.de` / `ns2.inwx.de` / `ns3.inwx.eu`
2. Query the parent DS directly (for `.de`, `@a.nic.de`)
3. If the zone is stuck with mismatched DS and no authoritative `DNSKEY`, a clean cycle of:

```text
dnssec.deleteall {domainName: <domain>}
dnssec.enablednssec {domainName: <domain>}
```

can re-trigger signing and DS publication cleanly

## Record-by-record sync is safer than zone import

If `nameserver.importZone` does not preserve all records, replace all non-`SOA` and non-`NS` records explicitly:

1. Read current INWX records with `nameserver.info`.
2. Delete all records except `SOA` and `NS`.
3. Recreate each desired rrset record individually with `nameserver.createRecord`.

This avoided missing records during a migration where apex `A/AAAA`, wildcard CNAMEs, and service aliases did not survive the bulk import.

## Record format quirks

INWX accepts standard record content, but a few types need care:

- `TXT`: store unquoted content
- `CNAME`: strip the trailing dot before upload
- `MX`: put the MX preference in `prio`, and only the hostname in `content`
- `SRV`: put the SRV priority in `prio`, and only `weight port target` in `content`

Example:

```text
name: _autodiscover._tcp.example.com
type: SRV
prio: 0
content: 0 443 mailbox.org
```

The full deSEC-style string `0 0 443 mailbox.org` in `content` is rejected by INWX.

## Proxmox ACME with INWX

Proxmox uses the acme.sh INWX provider and expects these variable names:

```text
INWX_User
INWX_Password
INWX_Shared_Secret
```

Useful locations:

- `/etc/pve/nodes/<node>/config`
- `/etc/pve/priv/acme/plugins.cfg`
- `/usr/share/proxmox-acme/dnsapi/dns_inwx.sh`

## Traefik / lego ACME with INWX

Traefik uses lego and expects different variable names:

```text
INWX_USERNAME
INWX_PASSWORD
INWX_SHARED_SECRET
```

Also update the DNS challenge provider:

```yaml
dnsChallenge:
  provider: inwx
  resolvers:
    - "ns.inwx.de:53"
    - "ns2.inwx.de:53"
    - "ns3.inwx.eu:53"
```

## DNSSEC validation checklist

After migration:

1. `dnssec.info` should report `AUTO`
2. INWX authoritative nameservers should serve DNSKEY records
3. The parent should publish DS records for the domain

Do not consider DNSSEC fully complete until the DS records are visible at the parent.

## References

- https://www.inwx.de/en/help/apicommands
- https://www.inwx.de/en/help/nameserver
- https://www.inwx.de/en/help/dnssec
- https://pve.proxmox.com/pve-docs/
- https://doc.traefik.io/traefik/https/acme/
