# AGENTS.md

This file defines behavioral rules for AI agents working in this environment. All reusable knowledge and learned patterns are stored in the [CQ knowledge base](https://cq-mcp.bartschnet.de).

## Repository Contents

This repository (`~/.copilot/skills/`) contains:

- **`AGENTS.md`** — agent behavioral rules (this file)
- **`copilot-instructions.md`** — session initialization and core directives
- **`hardware/`** — hardware reference files (manuals, BIOS ROMs, diagrams)

All skills, procedures, and learned patterns belong in **CQ** — not in this repository.

## Document Quirks, Roadblocks, and API Structures

Whenever you encounter unexpected behavior, a non-obvious limitation, or a working pattern through trial and error, **immediately store it in CQ via `cq-propose`**. Do not wait until the end of the session.

Specifically, always capture:

- **API quirks**: required fields not obvious from the docs, silently ignored fields, misleading error messages, or undocumented defaults (e.g., "PUT /api/groups requires `name` even when only changing peers, else returns 422").
- **Roadblocks and failure modes**: things that look like they should work but don't, and the actual root cause (e.g., "Netbird Networks subnet resources do NOT install OS-level routes on client peers — use classic routes instead").
- **Working patterns**: the exact sequence of steps or API calls that solved a problem, so the same effort is not repeated.
- **Non-obvious dependencies**: services or configs that must be in a certain state before a step will succeed.

Use `⚠️` in the `detail` field to mark critical pitfalls that would cause hard-to-diagnose failures.

The goal is to avoid duplicating debugging effort across sessions. If you wasted time on something, capture it so future sessions don't repeat the same journey.

## Network Overview Before Any Changes

**Before planning or making any change to the network** (routing, DNS, firewall rules, VPN peers, IP assignments, etc.), always get an overview of the current network structure first:

1. Query CQ with relevant domain tags (e.g., `weissdornweg`, `friedensstrasse`, `netbird`, `proxmox`, `docker`) to retrieve the current topology, IP assignments, and service locations.
2. If live state is needed, query the relevant systems directly (e.g., Netbird API for peers/groups/policies, CoreDNS config, Traefik routing rules).
3. Only after understanding the current state, plan and execute changes.

This prevents unintended side effects from stale assumptions (e.g., deleting a peer that is still a router, reconfiguring DNS without knowing what depends on it, or introducing routing conflicts).

## Security Rules

**This is a public GitHub repository. Never commit passwords, tokens, API keys, or other secrets into any file.** If an example requires credentials, use placeholder values like `<your-token>`, `<api-key>`, or `$TOKEN` (shell variable).

This rule applies to:
- API tokens (e.g. NetBird `nbp_...`, GitHub PATs, cloud provider keys)
- Passwords and passphrases
- Private keys or certificates
- Any value that would grant access if disclosed

If a skill currently contains a real credential, remove it immediately, rotate the credential, and replace with a placeholder.

## Session Learning

At the end of every session, store new knowledge in CQ:

1. Call `cq-reflect` with a summary of the session context to surface candidate knowledge units.
2. Call `cq-propose` for each genuinely useful insight not already stored.
3. If the session updated `AGENTS.md` or `copilot-instructions.md`, run `git -C ~/.copilot/skills add -A && git -C ~/.copilot/skills commit -m "<message>"` and push.

Apply this rule at the end of any session in which you worked with systems, tools, or configurations.

**Before context compaction:** Immediately follow steps 1–2 above to persist all accumulated learnings before they are lost.

