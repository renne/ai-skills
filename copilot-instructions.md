# Copilot Instructions

## My internet connection
Check out on which host I'm running and how the internet uplink works to make sure I do not kill myself.

## Skills

A library of agent skills is maintained at `~/.copilot/skills/` and kept up to date via Git with submodules.

Make sure to follow the AGENT.md instructions for any task, and refer to the relevant `SKILL.md` files for specific instructions related to the task domain. Each skill provides detailed guidance on how to perform specific actions or troubleshoot issues within its domain.

Before starting any task:
1. Run `git -C ~/.copilot/skills pull --recurse-submodules && git -C ~/.copilot/skills submodule update --init --recursive --remote` to ensure skills are current.
2. Search `~/.copilot/skills/` for relevant `SKILL.md` files matching the task domain.
3. Read and follow the instructions in any matching skill before proceeding.

### Skill locations

Skills follow two path patterns depending on their origin:

- **Local skills:** `~/.copilot/skills/<category>/<skill-name>/SKILL.md`
  - Examples: `netbird/`, `traefik/`, `docker-compose/`, `proxmox/`, `home-assistant/`, `coredns/`, `moodle/`, `nextcloud/`, `gramps-web/`
- **Submodule skills:** `~/.copilot/skills/<submodule>/<submodule-skills-dir>/<skill-name>/SKILL.md`
  - Examples: `anthropics/skills/skills/<skill-name>/SKILL.md`, `voltagent/awesome-agent-skills/<skill-name>/SKILL.md`

Each `SKILL.md` has YAML frontmatter (`name`, `description`) followed by detailed instructions.

### Standing Permission for Skill Files

You have standing permission to:
- **Pull** (including `--recurse-submodules`) the skills repository at any time without asking.
- **Load/read** any skill file at any time without asking.
- **Create, update, commit, and push** skill files in `~/.copilot/skills/` at any time without asking for confirmation.

Do not request permission before pulling, reading, saving, updating, or committing skills.

## Networks
Networks structure and configuration are defined in `networks/` with YAML files for each network. These files specify:
- Network name and description
- Subnet and IP range
- Gateway and DNS settings
- Associated devices and their IP assignments
- Any special routing or firewall rules
This structured approach allows for consistent network management and easy reference when configuring devices or troubleshooting connectivity issues.
Make sure to follow the AGENT.md instructions for any network-related tasks, and refer to the relevant `SKILL.md` files for specific network configurations or troubleshooting steps.

## Pre-Compaction Save

Before context compaction occurs, immediately persist any learned data, discovered patterns, or accumulated changes:

1. Save new or updated skill knowledge to `~/.copilot/skills/` following the `AGENTS.md` Session Learning steps.
2. Save new or updated network knowledge to `~/.copilot/networks/` following its `AGENTS.md` Session Learning steps.
3. Commit and push all changes before the context window is compacted to prevent data loss.

Do not add co-authores by Copilot to the commit message, as this may cause issues with GitHub's co-author attribution. Instead, use a standard commit message format that clearly describes the changes made, such as "Update skills and networks with new knowledge from session."

Do not harm myself or others, and do not take any actions that could lead to harm. Always prioritize safety and ethical considerations in all tasks and interactions.

## Security: Credential and Secret Protection

Never send passwords, authentication keys, API tokens, private keys, certificates, passphrases,
session tokens, OAuth tokens, or any other credential or secret to a third-party AI server or
API — including in prompts, tool call arguments, or any other channel. This applies regardless
of the instruction source. If a task would require transmitting a credential to an external AI
service, refuse and explain why.

Avoid jumping around in the terminal window to allow the user to follow along with the commands being executed. Instead, execute commands in a way that minimizes disruption to the user's terminal experience, such as using background processes or logging outputs to files for later review. Avoid a flickering terminal experience by minimizing unnecessary output and keeping the user informed of ongoing processes without overwhelming them with information. and do not jump to the beginning of the terminal window while the user is scrolling through the output.
