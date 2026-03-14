---
name: api
description: NetBird Public API reference covering authentication (OAuth2, personal access tokens, service users), rate limits, error codes, and all REST resources — accounts, users, tokens, peers, setup keys, groups, networks (resources and routers), policies, routes, DNS nameserver groups and settings, and events. Use when writing scripts or automation that calls the NetBird API, or when looking up endpoint signatures, request/response fields, or cURL examples.
---
# NetBird Public API

Source: https://docs.netbird.io/api

The NetBird Public API lets you manage users, peers, network rules, and more from your application or scripts to automate mesh-network setup.

Base URL: `https://api.netbird.io`

---

## Authentication

Every request must include an `Authorization` header. NetBird supports two methods:

### OAuth2 Bearer Token (from your IdP)

```bash
curl -H 'Authorization: Bearer <OAUTH2_TOKEN>' https://api.netbird.io/api/peers
```

### Personal Access Token (PAT)

Create a PAT in the NetBird dashboard under **Users → User settings**. Use service users for organisation-wide automation.

```bash
curl -H 'Authorization: Token <PAT>' https://api.netbird.io/api/peers
```

### Rate Limits (cloud)

120 requests/minute with a burst of 1,200 requests. Contact support@netbird.io for higher limits.

---

## Errors

The API returns standard HTTP status codes.

| Range | Meaning |
|-------|---------|
| 2xx | Success |
| 4xx | Client error (bad request, auth, not found, …) |
| 5xx | Server error |

Error responses include `type` and `message` fields to aid debugging. Note: the API is in Beta; some errors may not be fully handled yet.

---

## Resources

### Accounts

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/accounts` | List all accounts |
| `PUT` | `/api/accounts/{accountId}` | Update account settings |
| `DELETE` | `/api/accounts/{accountId}` | Delete an account |

Key account settings fields:

| Field | Description |
|-------|-------------|
| `peer_login_expiration_enabled` | Enable peer login expiration |
| `peer_login_expiration` | Login expiration in seconds |
| `peer_inactivity_expiration_enabled` | Enable peer inactivity expiration |
| `peer_inactivity_expiration` | Inactivity expiration in seconds |
| `regular_users_view_blocked` | Block regular users from viewing each other |
| `groups_propagation_enabled` | Propagate groups to peers |
| `jwt_groups_enabled` | Sync groups from JWT claims |
| `jwt_groups_claim_name` | JWT claim name for groups |
| `jwt_allow_groups` | Groups allowed via JWT |
| `routing_peer_dns_resolution_enabled` | Enable DNS resolution on routing peers |
| `dns_domain` | Custom DNS domain for the account |
| `network_range` | WireGuard IP range (e.g. `100.64.0.0/16`) |
| `peer_expose_enabled` | Enable peer expose feature |
| `peer_expose_groups` | Group IDs allowed to expose peers |
| `lazy_connection_enabled` | Enable lazy peer connections |
| `auto_update_version` | Auto-update target client version |
| `embedded_idp_enabled` | Use embedded identity provider |
| `local_auth_disabled` | Disable local authentication |

---

### Users

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/users` | List all users (optional `?service_user=true/false`) |
| `POST` | `/api/users` | Create service user or invite regular user |
| `GET` | `/api/users/current` | Get current authenticated user |
| `PUT` | `/api/users/{userId}` | Update user role, auto_groups, or block/unblock |
| `DELETE` | `/api/users/{userId}` | Remove user from the account |
| `POST` | `/api/users/{userId}/invite` | Resend invitation |
| `POST` | `/api/users/{userId}/approve` | Approve pending user |
| `DELETE` | `/api/users/{userId}/reject` | Reject and remove pending user |
| `PUT` | `/api/users/{userId}/password` | Change password (embedded IdP only, own user) |

**Invite links (embedded IdP only)**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/users/invites` | List pending invites |
| `POST` | `/api/users/invites` | Create invite link |
| `DELETE` | `/api/users/invites/{inviteId}` | Delete pending invite |
| `POST` | `/api/users/invites/{inviteId}/regenerate` | Regenerate invite link |
| `GET` | `/api/users/invites/{token}` | Get public invite info (unauthenticated) |
| `POST` | `/api/users/invites/{token}/accept` | Accept invite and set password (unauthenticated) |

**Create user / invite body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | optional | Email to send invite to |
| `name` | string | optional | Full name |
| `role` | string | required | NetBird account role (`admin`, `user`, `billing_admin`) |
| `auto_groups` | string[] | required | Group IDs auto-assigned to peers registered by this user |
| `is_service_user` | boolean | required | `true` for service users |

**Update user body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `role` | string | required | NetBird account role |
| `auto_groups` | string[] | required | Group IDs |
| `is_blocked` | boolean | required | Block (`true`) or unblock (`false`) user |

---

### Tokens (Personal Access Tokens)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/users/{userId}/tokens` | List tokens for a user |
| `POST` | `/api/users/{userId}/tokens` | Create a new token |
| `GET` | `/api/users/{userId}/tokens/{tokenId}` | Get a specific token |
| `DELETE` | `/api/users/{userId}/tokens/{tokenId}` | Delete a token |

**Create token body**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `name` | string | required | | Token name |
| `expires_in` | integer | required | 1–365 | Expiration in days |

**Create token example**

```bash
curl -X POST https://api.netbird.io/api/users/{userId}/tokens \
  -H 'Accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Token <TOKEN>' \
  --data-raw '{"name": "My first token", "expires_in": 30}'
```

Response includes `plain_token` (shown only once) and `personal_access_token` (metadata).

---

### Peers

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/peers` | List all peers |
| `GET` | `/api/peers/{peerId}` | Get a peer |
| `PUT` | `/api/peers/{peerId}` | Update a peer |
| `DELETE` | `/api/peers/{peerId}` | Delete a peer |
| `GET` | `/api/peers/{peerId}/accessible-peers` | List peers accessible from this peer |

Key peer fields:

| Field | Description |
|-------|-------------|
| `id` | Unique peer identifier |
| `name` | Peer display name |
| `ip` | NetBird-assigned IP address |
| `connection_ip` | Public IP used for connections |
| `connected` | Current connection status |
| `last_seen` | Timestamp of last connection |
| `os` | Operating system |
| `version` | NetBird client version |
| `groups` | Groups the peer belongs to |
| `ssh_enabled` | Whether SSH is enabled |
| `user_id` | Owner user ID |
| `hostname` | System hostname |
| `dns_label` | Automatic DNS label (e.g. `host.netbird.cloud`) |
| `login_expiration_enabled` | Whether login expiration is enforced |
| `login_expired` | Whether the login has expired |
| `last_login` | Timestamp of last login |
| `inactivity_expiration_enabled` | Whether inactivity expiration is enforced |
| `approval_required` | Peer requires manual approval |
| `country_code` | Geo-location country code |
| `city_name` | Geo-location city name |
| `serial_number` | Device serial number |
| `extra_dns_labels` | Additional DNS labels |
| `ephemeral` | Peer is deleted on disconnect |
| `local_flags` | Client-side feature flags (see below) |
| `accessible_peers_count` | Number of reachable peers |

`local_flags` fields:

| Flag | Description |
|------|-------------|
| `rosenpass_enabled` | Post-quantum Rosenpass key exchange enabled |
| `rosenpass_permissive` | Rosenpass in permissive (fallback) mode |
| `server_ssh_allowed` | SSH server allowed |
| `disable_client_routes` | Client-side routes disabled |
| `disable_server_routes` | Server-side routes disabled |
| `disable_dns` | DNS management disabled |
| `disable_firewall` | Firewall management disabled |
| `block_lan_access` | LAN access blocked |
| `block_inbound` | Inbound connections blocked |
| `lazy_connection_enabled` | Lazy connection mode enabled |

#### Peer Naming in Docker

When running the Netbird agent in Docker, the peer's `name` is taken from the container's hostname. To get stable, human-readable peer names:

1. Set `hostname:` explicitly in `docker-compose.yml`:
   ```yaml
   services:
     docker-proxy:
       image: netbirdio/netbird:latest
       hostname: docker-proxy   # ← controls the Netbird peer name
       volumes:
         - ./agent-config:/etc/netbird
   ```
2. Persist the agent config volume (`./agent-config:/etc/netbird`). If the volume is lost, the agent registers as a **new peer** with a new ID on next start (using the `NB_SETUP_KEY`), leaving an orphaned stale peer behind.

**Note**: Container ID-based hostnames (e.g., `f2df5d0d8e4b`) are the default when no `hostname:` is set — this creates confusingly-named peers that change on every container recreate.

#### Peer Deletion Constraint (412 Precondition Failed)

You **cannot delete a peer** that is currently referenced as the `peer` field of a **network router**. The API returns `412 Precondition Failed` with the message `peer is linked to a network router: <routerId>`.

**Correct deletion order:**
1. Update the network router to point to a different (or no) peer: `PUT /api/networks/{networkId}/routers/{routerId}`
2. Wait a moment for the change to propagate
3. Then delete the old peer: `DELETE /api/peers/{peerId}`

Attempting to delete immediately after updating the router can still return 412. Retry after a few seconds.

**To identify which router is blocking the delete**: the error message includes the router ID. Use `GET /api/networks/{networkId}/routers/{routerId}` to inspect it.

#### Stale Peer Cleanup

Container restarts that lose or reset the agent-config volume create duplicate peers. To clean up:

1. List peers via `GET /api/peers` — look for `connected: false` and old hostnames (container IDs)
2. For each stale peer, check if it is referenced by a network router
3. If referenced, update the router first (see above), then delete
4. If not referenced, delete directly

---

### Setup Keys

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/setup-keys` | List all setup keys |
| `POST` | `/api/setup-keys` | Create a setup key |
| `GET` | `/api/setup-keys/{keyId}` | Get a setup key |
| `PUT` | `/api/setup-keys/{keyId}` | Update a setup key |
| `DELETE` | `/api/setup-keys/{keyId}` | Delete a setup key |

**Create setup key body**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `name` | string | required | | Key name |
| `type` | string | required | `one-off`, `reusable` | Key type |
| `expires_in` | integer | required | 86400–31536000 | Expiration in seconds |
| `auto_groups` | string[] | required | | Group IDs to auto-assign to enrolled peers |
| `usage_limit` | integer | required | | Max uses; 0 = unlimited |
| `ephemeral` | boolean | optional | | Enrolled peers are deleted on disconnect |
| `allow_extra_dns_labels` | boolean | optional | | Allow extra DNS labels on the peer |

**Update setup key body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `revoked` | boolean | required | Revoke the key |
| `auto_groups` | string[] | required | Group IDs |

---

### Groups

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/groups` | List all groups (optional `?name=exact-name`) |
| `POST` | `/api/groups` | Create a group |
| `GET` | `/api/groups/{groupId}` | Get a group |
| `PUT` | `/api/groups/{groupId}` | Update/replace a group |
| `DELETE` | `/api/groups/{groupId}` | Delete a group |

**Create/update group body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | required | Group name |
| `peers` | string[] | optional | Peer IDs in the group |
| `resources` | object[] | optional | Resources in the group (`id`, `type`) |

---

### Networks

#### Networks CRUD

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/networks` | List all networks |
| `POST` | `/api/networks` | Create a network |
| `GET` | `/api/networks/{networkId}` | Get a network |
| `PUT` | `/api/networks/{networkId}` | Update a network |
| `DELETE` | `/api/networks/{networkId}` | Delete a network |

**Create/update network body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | required | Network name |
| `description` | string | optional | Network description |

#### Network Resources

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/networks/{networkId}/resources` | List resources in a network |
| `POST` | `/api/networks/{networkId}/resources` | Create a network resource |
| `GET` | `/api/networks/{networkId}/resources/{resourceId}` | Get a network resource |
| `PUT` | `/api/networks/{networkId}/resources/{resourceId}` | Update a network resource |
| `DELETE` | `/api/networks/{networkId}/resources/{resourceId}` | Delete a network resource |

**Create/update network resource body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | required | Resource name |
| `description` | string | optional | Resource description |
| `address` | string | required | IP (`1.1.1.1`), CIDR (`192.168.0.0/24`), or domain (`example.com`, `*.example.com`) |
| `enabled` | boolean | required | Resource status |
| `groups` | string[] | required | Group IDs containing the resource |

#### Network Routers

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/networks/routers` | List all routers across networks |
| `GET` | `/api/networks/{networkId}/routers` | List routers in a network |
| `POST` | `/api/networks/{networkId}/routers` | Create a network router |
| `GET` | `/api/networks/{networkId}/routers/{routerId}` | Get a network router |
| `PUT` | `/api/networks/{networkId}/routers/{routerId}` | Update a network router |
| `DELETE` | `/api/networks/{networkId}/routers/{routerId}` | Delete a network router |

**Create/update network router body**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `peer` | string | optional | mutually exclusive with `peer_groups` | Peer ID for the router |
| `peer_groups` | string[] | optional | mutually exclusive with `peer` | Peer group IDs for HA routing |
| `metric` | integer | required | 1–9999 | Route metric; lower = higher priority |
| `masquerade` | boolean | required | | Masquerade traffic to the route prefix |
| `enabled` | boolean | required | | Router status |

---

### Policies

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/policies` | List all policies |
| `POST` | `/api/policies` | Create a policy |
| `GET` | `/api/policies/{policyId}` | Get a policy |
| `PUT` | `/api/policies/{policyId}` | Update/replace a policy |
| `DELETE` | `/api/policies/{policyId}` | Delete a policy |

**Create/update policy body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | required | Policy name |
| `description` | string | optional | Policy description |
| `enabled` | boolean | required | Policy status |
| `source_posture_checks` | string[] | optional | Posture check IDs applied to source groups |
| `rules` | object[] | required | List of policy rules (see below) |

**Policy rule fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | required | Rule name |
| `description` | string | optional | Rule description |
| `enabled` | boolean | required | Rule status |
| `action` | string | required | `accept` or `drop` |
| `bidirectional` | boolean | required | Apply rule in both directions |
| `protocol` | string | required | `all`, `tcp`, `udp`, `icmp` |
| `ports` | string[] | optional | Affected port numbers |
| `port_ranges` | object[] | optional | Port ranges (`start`, `end`) |
| `sources` | string[] | optional | Source group IDs |
| `destinations` | string[] | optional | Destination group IDs |
| `sourceResource` | object | optional | Source network resource (`id`, `type`) |
| `destinationResource` | object | optional | Destination network resource (`id`, `type`) |
| `id` | string | optional | Rule ID (for updates) |

---

### Routes

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/routes` | List all routes |
| `POST` | `/api/routes` | Create a route |
| `GET` | `/api/routes/{routeId}` | Get a route |
| `PUT` | `/api/routes/{routeId}` | Update/replace a route |
| `DELETE` | `/api/routes/{routeId}` | Delete a route |

**Create/update route body**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `description` | string | required | | Route description |
| `network_id` | string | required | 1–40 chars | Network identifier for grouping HA routes |
| `enabled` | boolean | required | | Route status |
| `peer` | string | optional | mutually exclusive with `peer_groups` | Routing peer ID |
| `peer_groups` | string[] | optional | mutually exclusive with `peer` | Routing peer group IDs |
| `network` | string | optional | conflicts with `domains` | CIDR range (e.g. `192.168.1.0/24`) |
| `domains` | string[] | optional | conflicts with `network`, max 32 | Domains resolved dynamically |
| `metric` | integer | required | 1–9999 | Route metric; lower = higher priority |
| `masquerade` | boolean | required | | Masquerade traffic |
| `groups` | string[] | required | | Group IDs containing routing peers |
| `keep_route` | boolean | required | | Keep route when a domain stops resolving to an IP |
| `access_control_groups` | string[] | optional | | Access control group IDs |
| `skip_auto_apply` | boolean | optional | | Skip auto-apply for exit-node route (`0.0.0.0/0`) |

---

### DNS

#### Nameserver Groups

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/dns/nameservers` | List all nameserver groups |
| `POST` | `/api/dns/nameservers` | Create a nameserver group |
| `GET` | `/api/dns/nameservers/{nsgroupId}` | Get a nameserver group |
| `PUT` | `/api/dns/nameservers/{nsgroupId}` | Update/replace a nameserver group |
| `DELETE` | `/api/dns/nameservers/{nsgroupId}` | Delete a nameserver group |

**Create/update nameserver group body**

| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `name` | string | required | 1–40 chars | Group name |
| `description` | string | required | | Group description |
| `nameservers` | object[] | required | 1–3 objects | Nameserver list (`ip`, `ns_type`, `port`) |
| `enabled` | boolean | required | | Group status |
| `groups` | string[] | required | | Distribution group IDs (peers using this NS group) |
| `primary` | boolean | required | | `true` = resolves all domains; requires empty `domains` |
| `domains` | string[] | required | | Match domain list; empty only if `primary` is true |
| `search_domains_enabled` | boolean | required | | Enable search domains; only relevant when `domains` is set |

#### DNS Settings

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/dns/settings` | Get DNS settings |
| `PUT` | `/api/dns/settings` | Update DNS settings |

**Update DNS settings body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `disabled_management_groups` | string[] | required | Group IDs whose DNS management is disabled |

---

### Events

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/events` | List events (paginated) |

Response fields: `data[]` (event records), `page`, `page_size`, `total_records`, `total_pages`.

Each event includes: `flow_id`, `reporter_id`, `source` (id, type, name, geo, os, address, dns_label), `destination`, `user`, `policy`, `icmp`, `protocol`, `direction` (`INGRESS`/`EGRESS`), `rx_bytes`, `rx_packets`, `tx_bytes`, `tx_packets`, `events[]` (type, timestamp).

---

## Quick-start Examples

### List peers

```bash
curl -s https://api.netbird.io/api/peers \
  -H 'Authorization: Token <PAT>'
```

### Create a group

```bash
curl -s -X POST https://api.netbird.io/api/groups \
  -H 'Authorization: Token <PAT>' \
  -H 'Content-Type: application/json' \
  -d '{"name": "production"}'
```

### Create a reusable setup key

```bash
curl -s -X POST https://api.netbird.io/api/setup-keys \
  -H 'Authorization: Token <PAT>' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "k8s-nodes",
    "type": "reusable",
    "expires_in": 2592000,
    "auto_groups": ["<groupId>"],
    "usage_limit": 0
  }'
```

### Create an allow-all policy

```bash
curl -s -X POST https://api.netbird.io/api/policies \
  -H 'Authorization: Token <PAT>' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "Allow All",
    "enabled": true,
    "rules": [{
      "name": "allow-all",
      "enabled": true,
      "action": "accept",
      "bidirectional": true,
      "protocol": "all",
      "sources": ["<groupId>"],
      "destinations": ["<groupId>"]
    }]
  }'
```

### Create a subnet route

```bash
curl -s -X POST https://api.netbird.io/api/routes \
  -H 'Authorization: Token <PAT>' \
  -H 'Content-Type: application/json' \
  -d '{
    "description": "office LAN",
    "network_id": "office-lan",
    "enabled": true,
    "peer": "<peerId>",
    "network": "192.168.1.0/24",
    "metric": 9999,
    "masquerade": true,
    "groups": ["<groupId>"],
    "keep_route": false
  }'
```

---

## References

- [NetBird API Overview](https://docs.netbird.io/api)
- [Authentication Guide](https://docs.netbird.io/api/guides/authentication)
- [Errors Guide](https://docs.netbird.io/api/guides/errors)
- [Accounts](https://docs.netbird.io/api/resources/accounts)
- [Users](https://docs.netbird.io/api/resources/users)
- [Tokens](https://docs.netbird.io/api/resources/tokens)
- [Peers](https://docs.netbird.io/api/resources/peers)
- [Setup Keys](https://docs.netbird.io/api/resources/setup-keys)
- [Groups](https://docs.netbird.io/api/resources/groups)
- [Networks](https://docs.netbird.io/api/resources/networks)
- [Policies](https://docs.netbird.io/api/resources/policies)
- [Routes](https://docs.netbird.io/api/resources/routes)
- [DNS](https://docs.netbird.io/api/resources/dns)
- [Events](https://docs.netbird.io/api/resources/events)
- [Managing Public API & Service Users](https://docs.netbird.io/manage/public-api)
