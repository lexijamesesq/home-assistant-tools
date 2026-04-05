---
tags:
  - type/claude-project
  - project/home-assistant
status: active
description: "Home Assistant server maintenance, modernization, and automation management."
---
# Home Assistant

---

## Project State

**Last Updated:** TODO: date

### Re-entry Cue
TODO: One sentence — what should the next session pick up?

### Current State
- Core: TODO: version
- OS: TODO: version
- Supervisor: TODO: version
- Repairs: TODO: count and description

### Next Actions (Max 5)
1. TODO: First priority item
2. TODO: Second priority item

### Waiting For
- TODO: External blockers

### Decisions Needed
- TODO: Questions blocking progress

---

## Configuration

Instance-specific values that skills reference by config key rather than hardcoding.

```yaml
# SSH access to HA instance
ssh.host: "root@YOUR_HA_IP"
ssh.key: "~/.ssh/YOUR_KEY_NAME"
ssh.config_path: "/config"

# HA instance URLs
ha.local_url: "http://homeassistant.local:8123"

# Add-on identifiers (Supervisor CLI slug format)
# Find yours with: ssh user@ha "ha apps list | grep mcp"
addons.mcp_server_slug: "YOUR_MCP_ADDON_SLUG"

# Dashboard (default views for visual-diff captures)
dashboard.name: "YOUR_DASHBOARD_PATH"
dashboard.views:
  - home
  - TODO: your view names
```

Referenced by: `/ha-update`, `/theme-update`, `/visual-diff`

---

## Tool Selection Rules

- **Entity states/services:** HA MCP server for all state queries, service calls, and automation inspection.
- **YAML config:** SSH (using connection details from Configuration > `ssh.*`) for reading/editing configuration files.
- **Before investigating any system:** Read relevant Context/ docs from Key Files below first.

---

## Key Files

| File | Purpose |
|------|---------|
| `backlog.json` | Task tracking (phased: audit, stabilize, update, modernize, expand) |
| `backlog-archive.json` | Completed/cancelled items (moved during /session-closeout) |
| `progress.md` | Append-only session log |
| `Context/` | Rich context docs for backlog items |
| `Knowledge/` | Best practices and patterns for future reference |
| `Eval/` | Repeatable test suites for validating HA changes |

---

## System Info

| Property | Value |
|----------|-------|
| Platform | TODO: your hardware |
| Install Method | Home Assistant OS |
| Core | TODO: version |
| Internal IP | TODO: your IP |
| Local URL | TODO: your URL |

### Installed Add-ons
- TODO: list your add-ons

---

## Intake

### Tasks
**Method:** backlog-json
**Location:** backlog.json
**Schema:** Minimal (id, title, description, status, source, created, context_doc)
**Context:** Context/
