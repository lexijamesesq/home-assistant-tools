Claude Code skills for managing a Home Assistant instance. Handles system updates, theme maintenance, and dashboard validation through SSH and the HA MCP server — designed to run from a Mac managing a remote HA OS installation.

## Installation

Clone the repo, then set up the Claude Code directory:

```
mv claude .claude
```

Copy the sample config and fill in your values:

```
cp CLAUDE.sample.md CLAUDE.md
```

### Required configuration

| Field | Location | What to set |
|-------|----------|-------------|
| `ssh.host` | CLAUDE.md > Configuration | SSH user@host for your HA instance (e.g., `root@10.0.40.20`) |
| `ssh.key` | CLAUDE.md > Configuration | Path to your SSH private key |
| `ssh.config_path` | CLAUDE.md > Configuration | HA config directory on the remote host (usually `/config`) |
| `ha.local_url` | CLAUDE.md > Configuration | Local URL for your HA instance (e.g., `http://homeassistant.local:8123`) |

### Optional configuration

| Field | Location | What to set |
|-------|----------|-------------|
| `addons.mcp_server_slug` | CLAUDE.md > Configuration | Supervisor slug for the MCP Server add-on (needed for the bootstrap workaround in `/ha-update`) |
| `dashboard.name` | CLAUDE.md > Configuration | Your primary dashboard URL path (e.g., `dashboard-home`) |
| `dashboard.views` | CLAUDE.md > Configuration | List of view names for `/visual-diff` screenshot captures |

### Dependencies

- **Home Assistant OS** with SSH access (Terminal & SSH add-on, key auth)
- **Home Assistant MCP Server** add-on installed and configured as a Claude Code MCP server
- **Chrome MCP** for browser automation (used by `/visual-diff` and `/theme-update`)

## What's Included

### Skills

| Skill | What it does |
|-------|-------------|
| `/ha-update` | Full update lifecycle — inventories pending updates, classifies risk using learned heuristics, determines safe execution order, installs updates with validation gates, and refines its heuristics after each run. Handles Core, add-ons, HACS, and OS updates. Runs autonomously. |
| `/theme-update` | Merges a new Catppuccin distribution release into a customized Mush theme fork, preserving aesthetic overrides and applying HA 2026.x compatibility fixes (ramp flip, modes injection removal). |
| `/visual-diff` | Captures before/after screenshots of dashboard views for visual validation. Used by `/theme-update` and available standalone for any UI change. |

### Supporting files (not tracked — create your own)

| File | What it does |
|------|-------------|
| `Knowledge/ha-update-heuristics.md` | Distilled patterns `/ha-update` learns from each run — risk overrides, dependency patterns, validation checks, known gotchas |
| `Knowledge/ha-theme-architecture.md` | Reference doc for the Catppuccin Mush theme — ramp direction, modes injection, override markers |
| `Eval/` | Test suites for validating HA subsystems after updates (brightness, Sonos, etc.) |
| `Context/` | Design docs and implementation specs for backlog items |

## Configuration

The system separates what you configure from what skills handle.

**You configure:**
- `CLAUDE.md` — SSH connection details, HA URLs, add-on slugs, dashboard views
- `Knowledge/ha-update-heuristics.md` — starts empty, grows as `/ha-update` learns your system

**Skills handle:**
- Update inventory, risk assessment, and dependency ordering
- Breaking change cross-referencing against your installed integrations
- Backup creation, update execution, and post-update validation
- Theme merge with ramp correction and override preservation
- Dashboard screenshot capture and comparison

See `CLAUDE.sample.md` for the full configuration contract with placeholder values.

## Usage

### Update management

```
/ha-update
```
Inventories all pending updates, assesses risk, determines safe order, executes with validation after each. Learns from each run to improve future assessments. No arguments needed — discovers everything dynamically.

### Theme maintenance

```
/theme-update
```
Run after HACS updates the Catppuccin distribution. Extracts the new base, applies the ramp flip and semantic mirroring, merges your Mush overrides, validates, deploys, and presents a visual diff.

### Visual validation

```
/visual-diff before theme-update-2.1.4
/visual-diff after theme-update-2.1.4
```
Captures dashboard screenshots for before/after comparison. Works with any UI change, not just themes.

## How It Works

The `/ha-update` skill uses an adaptive loop — **assess, act, re-assess, act** — rather than a linear pipeline. It re-inventories after Core updates (which can reveal new updates), handles mid-run discovery (migration tasks surfaced in release notes), and tracks all work items in a run-scoped backlog that's discarded after the run.

Risk classification starts with defaults by category (Core monthly = medium, HACS patch = minimal) and applies learned overrides from the heuristics file. After each run, the skill distills what it learned into refined heuristics — not raw history, but patterns like "Kiosk Mode major bumps are dependency maintenance" or "pyscript may fail to load on first restart after Core update, retry once."

The MCP Server add-on cannot be updated via its own tools (bootstrap problem). The skill detects this and routes through SSH using the Supervisor CLI instead.

## Customization

- **Different HA instance:** Update `CLAUDE.md` Configuration with your SSH details and URLs. The skills resolve everything at runtime.
- **Different dashboard:** Update `dashboard.name` and `dashboard.views` in CLAUDE.md Configuration.
- **Different theme:** The `/theme-update` skill is specific to the Catppuccin Mush fork and its ramp-flip workaround. If you use a different theme, this skill won't apply — but it demonstrates a pattern for managing customized theme forks.
- **Without Chrome MCP:** `/visual-diff` and the visual validation step in `/theme-update` won't work. The other skills and all update functionality operate via ha-mcp and SSH only.

## Security

Review skills before installing. They load into Claude's context and execute with your permissions. The skills use SSH to access your HA instance and can install updates, restart services, and commit to git. Audit the contents of `claude/skills/` before use.

## License

MIT. See [LICENSE](LICENSE).
