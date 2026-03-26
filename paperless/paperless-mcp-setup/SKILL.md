---
name: paperless-mcp-setup
description: How to deploy and configure the paperless-mcp-server MCP integration with paperless-ngx, including generating the API token, configuring the .env file, setting the correct SSE transport type in mcp.json, and troubleshooting startup and container naming issues. Use when setting up, diagnosing, or repairing the Paperless-ngx MCP server connection.
---
# paperless-mcp-setup — Paperless-ngx MCP Server Setup

## Overview

`paperless-mcp-server` exposes 42 MCP tools for managing a Paperless-ngx instance (search, tag, classify, etc.). It communicates with the Paperless-ngx REST API using a token and exposes an **SSE MCP transport** for agent clients.

## Prerequisites

- Paperless-ngx running (all containers healthy)
- API token for the paperless user (see below)
- `paperless-mcp-server` added to the Docker Compose stack

## Docker Compose Configuration

Add to `compose.yml`:

```yaml
paperless-mcp:
  image: ghcr.io/renne/paperless-mcp-server:latest
  restart: unless-stopped
  depends_on:
    webserver:
      condition: service_healthy
  environment:
    PAPERLESS_URL: http://webserver:8000
    PAPERLESS_TOKEN: ${PAPERLESS_TOKEN}
    MCP_TRANSPORT: http   # server-side label; actual transport is SSE
  ports:
    - "3002:3002"
```

> **Note:** `MCP_TRANSPORT: http` is the value the server image uses internally. The client-facing protocol is still SSE — this label does not affect how the client must connect.

## Generating the API Token

Run against the paperless webserver container:

```bash
docker exec <webserver-container> python manage.py drf_create_token <username>
```

Example:
```bash
docker exec paperless-webserver-1 python manage.py drf_create_token myuser
```

Copy the token printed to stdout.

> **Important:** Use the actual Django username, not `admin` (unless the admin account is named `admin`). Run `docker exec <webserver-container> python manage.py shell -c "from django.contrib.auth.models import User; print([u.username for u in User.objects.all()])"` to list users.

## Setting the Token in .env

In the Docker Compose project directory, add to `.env`:

```
PAPERLESS_TOKEN=<token-from-above>
```

The `.env` file is loaded automatically by Docker Compose. The value is substituted for `${PAPERLESS_TOKEN}` in `compose.yml`.

After editing `.env`, recreate the MCP container:

```bash
docker compose up -d paperless-mcp
```

## Configuring mcp.json (Client Side)

In `~/.config/Code/User/mcp.json` (VS Code / Copilot CLI), the entry **must use `"type": "sse"`**:

```json
"paperless-ngx": {
  "type": "sse",
  "url": "https://dms-mcp.example.com/mcp"
}
```

> **Common mistake:** Using `"type": "http"` causes a "Session not found" error. The server requires an active SSE session — the client must `GET /mcp` first to receive a `sessionId` via `event: endpoint`, then `POST /mcp?sessionId=...` for subsequent messages. The `"http"` type skips the GET step. Always use `"type": "sse"`.

### No client-side auth headers needed

The MCP server authenticates to Paperless internally using `PAPERLESS_TOKEN`. There is no `Authorization` header needed in `mcp.json` for the paperless-ngx entry.

## Startup Behavior

`paperless-mcp` depends on the webserver being healthy (`condition: service_healthy`). The webserver health check passes only after all migrations and startup tasks complete — this can take **60–90 seconds** after a restart. The MCP container will wait automatically; do not force-recreate repeatedly.

## Troubleshooting

### Container name collision after failed compose up

If `docker compose up` is interrupted mid-rename, the new container may be named with a truncated ID prefix (e.g., `efb759cf8680_paperless-paperless-mcp-1`). Fix:

```bash
docker rm -f <old-container-name>
docker compose up -d paperless-mcp
```

This happens because Docker compose marks the old container for removal by prefixing its name before creating the replacement; if the process is interrupted, the prefixed container remains.

### Verify token is set

```bash
docker inspect <mcp-container> | grep PAPERLESS_TOKEN
```

If the value is empty (`PAPERLESS_TOKEN=`), check the `.env` file. Docker Compose reads it at `up` time — changes require a container restart to take effect.

### Test SSE connection manually

```bash
curl -sN https://dms-mcp.example.com/mcp
```

Expected: stream starting with `event: endpoint` and `data: /mcp?sessionId=...`.

### Available tools (42 total)

`search_documents`, `get_document`, `update_document`, `list_tags`, `list_correspondents`, `list_document_types`, `create_tag`, `create_correspondent`, `create_document_type`, `download_document`, `bulk_update_documents`, `delete_document`, `get_document_suggestions`, `get_document_metadata`, `list_storage_paths`, `get_storage_path`, `create_storage_path`, `update_storage_path`, `delete_storage_path`, `list_custom_fields`, `get_custom_field`, `create_custom_field`, `update_custom_field`, `delete_custom_field`, `list_saved_views`, `get_saved_view`, `create_saved_view`, `update_saved_view`, `delete_saved_view`, `get_tag`, `update_tag`, `delete_tag`, `get_correspondent`, `update_correspondent`, `delete_correspondent`, `get_document_type`, `update_document_type`, `delete_document_type`, `list_tasks`, `acknowledge_task`, `get_statistics`, `get_logs`
