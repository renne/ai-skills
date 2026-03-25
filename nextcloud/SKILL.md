---
name: nextcloud-oidc-idp
description: Using Nextcloud as an OpenID Connect Identity Provider (IdP) for other applications
---

# Nextcloud as OIDC Identity Provider

Nextcloud can act as a full OIDC IdP via the **OIDC Provider** app (app ID: `oidc`).
This is distinct from the **OpenID Connect user backend** app (app ID: `user_oidc`) which
makes Nextcloud a *consumer* of an external IdP.

## Prerequisites

- Nextcloud with the **OIDC Provider** app installed and enabled
- Admin access to run `occ` commands
- The Nextcloud instance reachable at a public HTTPS URL

## Creating a Client

Use the `occ oidc:create` command (not `user_oidc:client` — that belongs to the consumer app):

```bash
docker exec -u www-data <nextcloud-container> php occ oidc:create \
  --name "My App" \
  --type confidential \
  --id <client-id-or-leave-blank-for-auto> \
  --uri "https://myapp.example.com/oauth2/callback" \
  --flow code \
  --signing-alg RS256
```

### Key parameters

| Parameter | Typical value | Notes |
|-----------|--------------|-------|
| `--type` | `confidential` | Use `public` only for SPAs without a backend |
| `--flow` | `code` | Authorization Code flow — correct for oauth2-proxy and server-side apps |
| `--signing-alg` | `RS256` | Most compatible; some apps need `HS256` — check the consumer's docs |
| `--uri` | Redirect URI | Must match exactly what the consumer sends; can be specified multiple times |

The command outputs `client_id` and `client_secret`. Store them immediately — the secret
is not retrievable afterwards.

## OIDC Discovery Endpoint

```
https://<nextcloud-host>/.well-known/openid-configuration
```

Standard claims available: `sub`, `name`, `email`, `preferred_username`, `groups`.

## Known Quirks

### `email_verified` is not set to `true`

Nextcloud OIDC tokens do **not** include `"email_verified": true` even for verified accounts.
Consumers that require this claim will fail.

**Workaround:** When configuring the consumer, disable the email verification check:
- **oauth2-proxy**: add flag `--insecure-oidc-allow-unverified-email=true`
- **Dex / other brokers**: configure `insecure_skip_email_verified: true`

### `X-Auth-Request-User` contains `preferred_username`, not email

When oauth2-proxy injects the authenticated user identity into upstream requests, the
`X-Auth-Request-User` header contains the Nextcloud **username** (e.g., `renne`), not the
email address. Design upstream auth logic accordingly.

### Client secret length

Auto-generated client secrets are long (64+ chars). Ensure the consumer's config accepts
secrets of arbitrary length (most do; some legacy apps have limits).

## Integrating with oauth2-proxy

Minimal oauth2-proxy flags for Nextcloud IdP:

```yaml
# docker-compose / environment
OAUTH2_PROXY_PROVIDER: oidc
OAUTH2_PROXY_OIDC_ISSUER_URL: https://<nextcloud-host>
OAUTH2_PROXY_CLIENT_ID: <client-id>
OAUTH2_PROXY_CLIENT_SECRET: <client-secret>
OAUTH2_PROXY_REDIRECT_URL: https://<app-host>/oauth2/callback
OAUTH2_PROXY_COOKIE_SECRET: <32-byte-base64>
OAUTH2_PROXY_EMAIL_DOMAINS: "*"
OAUTH2_PROXY_INSECURE_OIDC_ALLOW_UNVERIFIED_EMAIL: "true"   # required for Nextcloud
OAUTH2_PROXY_PASS_ACCESS_TOKEN: "true"
OAUTH2_PROXY_SET_XAUTHREQUEST: "true"   # injects X-Auth-Request-User / Email / Groups
OAUTH2_PROXY_UPSTREAMS: http://<backend>:<port>
```

Generate cookie secret: `python3 -c "import secrets,base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"` 

## Traefik ForwardAuth Pattern

To protect services behind Traefik using oauth2-proxy as a ForwardAuth middleware:

```yaml
# Traefik labels on the protected service
- "traefik.http.middlewares.oauth2.forwardauth.address=http://oauth2-proxy:4180/oauth2/auth"
- "traefik.http.middlewares.oauth2.forwardauth.trustForwardHeader=true"
- "traefik.http.middlewares.oauth2.forwardauth.authResponseHeaders=X-Auth-Request-User,X-Auth-Request-Email,X-Auth-Request-Groups"
- "traefik.http.routers.myapp.middlewares=oauth2"
```

**Important:** oauth2-proxy must also have its own Traefik router to serve `/oauth2/*` routes
(callback, sign-in page, etc.) — these must NOT be gated by the ForwardAuth middleware:

```yaml
# oauth2-proxy service labels
- "traefik.http.routers.oauth2.rule=Host(`myapp.example.com`) && PathPrefix(`/oauth2/`)"
- "traefik.http.routers.oauth2.middlewares="   # no ForwardAuth on oauth2 routes
```

## SSO Token Pattern (eliminating double login)

When an app has its own login screen but you want seamless SSO after oauth2-proxy auth,
implement a backend endpoint that issues an app-specific token from the `X-Auth-Request-User`
header:

```
Browser → GET /api/auth/sso-token
  → Traefik (TLS)
  → oauth2-proxy (validates OIDC cookie, injects X-Auth-Request-User)
  → app backend (reads header, issues app JWT)
  ← { token: "...", username: "renne" }
```

Key security requirements:
- The app backend must **not** trust `X-Auth-Request-*` headers from the public internet —
  only from oauth2-proxy on the internal Docker network
- oauth2-proxy strips externally-supplied `X-Auth-Request-*` headers before proxying

See `~/.copilot/skills/tools/cq/SKILL.md` for a complete implementation example (CQ app).

## Example: Real Deployment

**CQ knowledge commons at `https://cq.bartschnet.de`:**

```
Nextcloud OIDC client:
  name: cq
  type: confidential
  flow: code
  signing-alg: RS256
  redirect_uri: https://cq.bartschnet.de/oauth2/callback
  client_id: WBbjVxCQC7ua3pvlBKKiS9oeCd7WsKFXgRn2LA6kcjo0GqJW

oauth2-proxy flags (non-secret):
  --provider=oidc
  --oidc-issuer-url=https://bartschnet.de
  --redirect-url=https://cq.bartschnet.de/oauth2/callback
  --email-domains=*
  --insecure-oidc-allow-unverified-email=true
  --pass-access-token=true
  --set-xauthrequest=true
  --upstream=http://cq-team-ui:3000
```
