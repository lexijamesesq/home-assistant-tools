---
tags:
  - type/claude-repo
description: "Claude Code skills for managing a Home Assistant instance — system updates, theme maintenance, dashboard validation via SSH and the HA MCP server."
docs_home: "{workspace_root}/Projects/Home Assistant"
---

# home-assistant-tools

Three Claude Code skills (`/ha-update`, `/theme-update`, `/visual-diff`) for maintaining a self-hosted Home Assistant instance — designed to run from a Mac managing a remote HA OS installation over SSH and the HA MCP server. Public repo: a template other people fork and configure for their own instance, not just this operator's tooling.

## Setup

Clone the repo. `.claude/` ships tracked and committed — review its contents (see Security below) before opening the directory in Claude Code. Copy the instance config sample and fill in your own values:

```
cp .claude/instance.sample.md .claude/instance.md
```

See `.claude/instance.sample.md` for the full configuration contract (SSH connection, HA URLs, add-on slugs, dashboard views) with placeholder values.

## Configuration

Skills read instance-specific values from `.claude/instance.md`'s Configuration section by key name, not hardcoded — `ssh.host`, `ssh.config_path`, `ha.local_url`/`ha.internal_url`/`ha.external_url`/`ha.mcp_endpoint`, `addons.mcp_server_slug`, `dashboard.name`/`dashboard.views`. `.claude/instance.md` is gitignored — every fork fills in its own.

## Build / Test

No build step (skills are markdown, not compiled). Local checks before pushing:

```bash
shellcheck **/*.sh          # if any shell scripts are added
uvx ruff@latest check .     # if any Python is added
pre-commit run --all-files  # gitleaks-staged + the standard hook set
```

## CI

`.github/workflows/ci.yml`, required via the "Protect main" ruleset: `shellcheck` (`ludeeus/action-shellcheck`) and `gitleaks` (full outgoing PR-range scan via dotty's shared `setup-gitleaks` composite action, base rules only + `--redact` — public repo, the operator's PII ruleset never reaches CI). Both required to merge.

## Conventions

- Skills are self-contained SKILL.md files under `.claude/skills/{name}/` — no shared runtime beyond the Configuration keys above.
- Instance-specific values are always config keys, never hardcoded — a skill that hardcodes a path/URL breaks for every other fork.
- Commits: gitleaks-staged/-pre-push/-commit-msg (dotty's exported hooks) gate every commit and push locally; CI re-proves the outgoing PR range independently.

## Key Files

| File | Purpose |
|------|---------|
| `.claude/instance.sample.md` | Configuration contract template — copy to `.claude/instance.md` and fill in your instance's values |
| `.claude/skills/ha-update/` | `/ha-update` — full update lifecycle (inventory, risk classification, safe ordering, validation gates) |
| `.claude/skills/theme-update/` | `/theme-update` — merges new Catppuccin distribution releases into a customized Mush theme fork |
| `.claude/skills/visual-diff/` | `/visual-diff` — before/after dashboard screenshots for visual validation |
