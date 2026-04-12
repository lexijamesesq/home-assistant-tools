---
tags:
  - type/claude-project
  - project/home-assistant
status: active
description: "Home Assistant instance maintenance, automation, integration, and long-term system health."
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
- TODO: other load-bearing subsystem state

### Next Actions (Max 5)
1. TODO: First priority item
2. TODO: Second priority item

### Waiting For
- TODO: External blockers

### Decisions Needed
- TODO: Questions blocking progress

---

## Knowledge Sources & Prioritization

When I need information about the HA system, I consult these in order:

1. **Live state** → `ha-mcp` tools (`ha_get_state`, `ha_get_automation_traces`, `ha_get_logbook`, `ha_config_get_automation`, etc.). Query, never assume. Entity states, automation traces, and live config are always authoritative over any doc.

2. **HA platform best practices** → `ha-mcp` MCP resource `skill://home-assistant-best-practices/SKILL.md`. Load on demand before creating or editing automations, scripts, helpers, or dashboards. Has a Reference Files table — load only the entries matching the current task, never the whole skill.

3. **This instance's documented knowledge** → `Knowledge/` folder. Persistent docs on this specific HA system (system reference, lighting architecture, mode system, theme, Sonos, security posture, update heuristics, LIFX workflows, organizational reference).

4. **Active backlog-item context** → `Context/` folder. Per-item scratch docs for in-progress work only. Empty at steady state; created when a backlog item needs working context; deleted or decoupled when the item closes.

**Not in this hierarchy:** CLAUDE.md itself does not duplicate Knowledge/ content. It holds project state, sensitive configuration, tool selection rules, and pointers into the hierarchy above.

---

## Tool Selection Rules

- **Entity states & services:** `ha-mcp` for all state queries, service calls, and automation inspection.
- **YAML config files on disk:** SSH (using connection details from Configuration > `ssh.*`) for reading/editing `/config/*.yaml`.
- **Before modifying an automation:** read via `ha_config_get_automation`, mutate specific fields, write back via `ha_config_set_automation`. Never retype configs from context — retyping fabricates values.
- **Before investigating a subsystem:** read the relevant `Knowledge/` doc first.
- **Before deleting or simplifying a condition block:** `git log -p -5 -- <file>` to check recent history for load-bearing removals.

---

## Key Files

| File | Purpose |
|------|---------|
| `backlog.json` | Task tracking (phased: audit, stabilize, update, modernize, expand) |
| `backlog-archive.json` | Completed/cancelled items (moved during `/session-closeout`) |
| `progress.md` | Append-only session log |
| `Knowledge/` | Persistent HA-system documentation — see Knowledge Sources section |
| `Context/` | Per-backlog-item scratch docs only — not persistent knowledge |
| `Eval/` | Repeatable test suites |
| `claude/skills/` | Project-level skills: `ha-update`, `theme-update`, `visual-diff` |

---

## Configuration

Instance-specific values that skills reference by config key rather than hardcoding. Sensitive — kept here intentionally.

```yaml
# SSH access to HA instance
ssh.host: "root@YOUR_HA_IP"
ssh.key: "~/.ssh/YOUR_KEY_NAME"
ssh.config_path: "/config"

# HA instance URLs
ha.local_url: "http://homeassistant.local:8123"
ha.internal_url: "http://YOUR_HA_IP:8123"
ha.external_url: "https://YOUR_EXTERNAL_HOSTNAME"
ha.mcp_endpoint: "http://YOUR_HA_IP:9583/YOUR_MCP_SECRET_PATH"

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

## System Info

| Property | Value |
|----------|-------|
| Platform | TODO: your hardware |
| Install Method | Home Assistant OS |
| Internal IP | TODO: your IP |
| External URL | TODO: your external URL |
| VLAN | TODO: your network layout |

For installed add-ons, integration catalog, protocol stack, physical layout, and other instance details, see `Knowledge/system-reference.md`. Current software versions live in the Current State section above.

---

## Project Phases

| Phase | Status | Focus |
|-------|--------|-------|
| 0. Research Spike | TODO | Tooling selection, project setup |
| 1. Audit | TODO | Full health check |
| 2. Stabilize | TODO | Fix broken integrations, clear repairs |
| 3. Update | TODO | Core, add-ons, HACS updates |
| 4. Modernize | TODO | Optimize automations, prevent regressions |
| 5. Expand | TODO | New projects |

---

## Intake

### Tasks
**Method:** backlog-json
**Location:** `backlog.json`
**Schema:** Extended (id, title, description, status, phase, source, created, context_doc)
**Context:** `Context/` — per-backlog-item scratch only; persistent knowledge goes to `Knowledge/`
