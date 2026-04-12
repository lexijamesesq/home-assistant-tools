---
name: ha-update
description: >
  Triggers when the user says "update HA", "check for updates", "run updates",
  "/ha-update", or similar Home Assistant update/upgrade requests.
  Manages the full lifecycle: inventory, risk assessment, ordering, execution,
  validation, and heuristic refinement.
user_invokable: true
allowed-tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash(ssh:*)
  - Bash(date:*)
  - Bash(mktemp:*)
  - Bash(rm:*)
  - Bash(cat:*)
  - Bash(sleep:*)
  - WebFetch
  - mcp__ha-mcp__ha_get_addon
  - mcp__ha-mcp__ha_get_updates
  - mcp__ha-mcp__ha_call_service
  - mcp__ha-mcp__ha_check_config
  - mcp__ha-mcp__ha_backup_create
  - mcp__ha-mcp__ha_get_state
  - mcp__ha-mcp__ha_get_states
  - mcp__ha-mcp__ha_get_system_health
  - mcp__ha-mcp__ha_search_entities
  - mcp__ha-mcp__ha_get_logbook
  - mcp__ha-mcp__ha_list_services
  - mcp__ha-mcp__ha_get_entity
  - mcp__ha-mcp__ha_get_overview
  - mcp__ha-mcp__ha_config_info
  - mcp__ha-mcp__ha_get_integration
  - mcp__ha-mcp__read_resource
  - mcp__ha-mcp__ha_get_skill_home_assistant_best_practices
  - mcp__ha-mcp__ha_restart
---

# /ha-update — Home Assistant Update Management

Manages the full lifecycle of HA updates: inventory, risk assessment, dependency ordering, execution, validation, and heuristic refinement. The skill adapts to whatever updates are pending — nothing is hardcoded.

## Invocation

```
/ha-update
```

No arguments. The skill discovers what needs updating dynamically.

## Role

You are the update manager for a Home Assistant instance. You assess risk, determine safe ordering, execute updates with validation gates, and learn from each run. You operate conservatively — stop and report on any validation failure rather than pushing forward.

## Key Files

| File | Purpose |
|------|---------|
| `Knowledge/update-heuristics.md` | Distilled patterns — read on startup, refine at closeout |
| `Context/ha-043-update-management.md` | Architecture and design rationale |
| `progress.md` | Project progress log — append summary at closeout |
| `backlog.json` | Project backlog — update status at closeout |

## System Access

SSH connection details and add-on identifiers are configured in the project CLAUDE.md under Configuration. Resolve these values at runtime by reading CLAUDE.md:
- SSH host: `Configuration > ssh.host`
- SSH key: `Configuration > ssh.key`
- HA config path on remote: `Configuration > ssh.config_path`
- MCP Server add-on slug: `Configuration > addons.mcp_server_slug`

## Execution Flow

The core loop is **assess → act → re-assess → act** until the run backlog is clear. This is not a linear pipeline — the skill re-inventories after significant updates and handles mid-run discovery as first-class work items.

### Phase A: Setup

1. **Load heuristics.** Read `Knowledge/update-heuristics.md`. These patterns inform risk classification, ordering, validation, and checkpoint decisions throughout the run.

2. **Load system config.** Read CLAUDE.md Configuration section to resolve SSH connection details and add-on identifiers for use throughout the run.

3. **Create run-scoped working files.** Create a temp directory for this run. Initialize two files:
   - `run-backlog.json` — tracks all work items (updates, migrations, discovered tasks). Schema:
     ```json
     {
       "run_id": "<ISO timestamp>",
       "baseline": {
         "repairs_count": null,
         "update_count": null,
         "system_health": null
       },
       "items": []
     }
     ```
   - `run-progress.md` — chronological narrative of the run. Start with a header and setup entry.

   Item schema for the run backlog:
   ```json
   {
     "id": "<type>-<n>",
     "type": "update | migration | task",
     "entity_id": "<update entity_id or null>",
     "title": "<human-readable description>",
     "from_version": "<current or null>",
     "to_version": "<target or null>",
     "category": "core | addon | hacs | os | supervisor | app",
     "risk": "minimal | low | medium | high",
     "status": "pending | approved | in-progress | completed | failed | deferred",
     "depends_on": "<item id or null>",
     "source": "initial-inventory | re-inventory | discovered-in-changelog | discovered-in-validation",
     "notes": "<findings, issues, or null>"
   }
   ```

4. **Capture baseline.** Before any changes:
   - Repairs count (note current number)
   - System health via `ha_get_system_health()`
   - Current update entity states via `ha_get_updates()`
   - **Log error snapshot** — `ha_get_logbook()` filtered for errors in the last hour. Record as the pre-update error baseline so post-update log scans can distinguish new errors from pre-existing ones.
   - **Integration state snapshot** — `ha_get_integration()` to record which integrations are already in `not_loaded`/`setup_error` states (intentional ignores vs problems). Used to diff against post-update state.
   - **Eval inventory** — scan `Eval/` directory for available test suites. Note their topics for matching against release note content later.
   - Record baseline in run backlog

### Phase B: Assess

5. **Inventory updates.** Call `ha_get_updates()`. For each update entity:
   - Determine category (core, addon, hacs, os, supervisor, app) from entity_id patterns and metadata
   - Classify risk using the heuristics risk classification rules:
     - Start with the default for the category
     - Check for heuristic overrides (component-specific patterns)
     - For Core: pull release notes with `ha_get_updates(entity_id=..., include_release_notes=true)`. Cross-reference `breaking_changes.entries` against `installed_integrations`. If any breaking change affects an installed integration, elevate risk to high.
     - For HACS major bumps: fetch changelog from `release_url` via WebFetch. Analyze content — if it's dependency maintenance, override risk downward. If it contains migration steps, keep medium or elevate.
     - Apply changelog interpretation heuristics (dependency bumps = low, "farewell" = check installed, etc.)
   - Check Core-responsiveness: compare release dates and changelogs of non-Core updates against the Core release date. If responsive, order after Core.
   - Detect dependencies:
     - Explicit: release notes mention minimum HA version requirements
     - Heuristic: ordering patterns from heuristics file
     - Implicit: components that fix version-specific bugs follow that version
   - Add to run backlog as a `pending` item

6. **Determine execution order.** Sort items by:
   - Dependency constraints (hard — must be respected)
   - Risk gradient from heuristics (soft — lowest risk first, highest last)
   - Category grouping from heuristics ordering heuristic

7. **Log assessment** to run progress with rationale for risk classifications and ordering decisions.

### Phase C: Execute (loop)

8. **Create backup via SSH and wait for completion.** Do NOT use `ha_backup_create()` (MCP) — it includes all add-ons with no exclusion option and times out at 120s. Do NOT use `update.install`'s `backup: true` flag (includes database, slow).

   Use SSH to create a selective partial backup that excludes slow or unnecessary add-ons.

   **Step 8a: Build the add-on list dynamically.** Query installed add-ons via `ha_get_addon()` (or SSH `ha apps info --raw-json`). Collect all installed add-on slugs. Then remove any slugs listed in the exclusion set below. The remaining slugs form the `--app` arguments for the backup command.

   **Backup exclusions** (configured in CLAUDE.md or heuristics — check both):
   - `a0d7b954_influxdb` — historical time-series data only, 15-16 min backup time, no rollback value for updates. Daily automatic backups cover it.

   If new add-ons have been installed since the last run, they will be automatically included. If add-ons have been removed, they won't appear in the query. Log the final add-on list in run progress so the record is clear.

   **Step 8b: Create the backup.**
   ```
   ssh -i {ssh.key} {ssh.host} "ha backups new --name 'Pre_Update_Run_<date>' --homeassistant-exclude-database --app <slug1> --app <slug2> ..."
   ```

   The `--app` flag triggers a partial backup with only the specified add-ons. The `--homeassistant-exclude-database` flag excludes the recorder database for speed. HA config is included by default.

   **The backup MUST complete before any updates begin.** The backup freezes the Supervisor while backing up add-ons. This freeze blocks OS updates, Supervisor restarts, and recovery commands. Proceeding with updates during an active backup risks I/O contention that can stall the backup indefinitely and leave the system frozen with no graceful recovery.

   The SSH command should return when the backup completes (no 120s MCP timeout). If it times out or hangs, verify via:
   ```
   ssh -i {ssh.key} {ssh.host} "ha jobs info" | grep -A 8 "<backup_job_uuid>"
   ```
   Poll until the top-level backup job shows `done: true`.

   **If backup gets stuck (>10 minutes without InfluxDB):** The system is frozen and recovery options are limited (`thaw` blocked by active job, `supervisor restart` blocked by freeze). Do NOT proceed with updates. Log the issue, note that daily automatic backups provide coverage, and defer the run.

   After backup completes, verify the system state is `running` (not `freeze`):
   ```
   ssh -i {ssh.key} {ssh.host} "ha info | grep state"
   ```

   Log backup completion to run progress, then proceed.

9. **Execute each item in order.** For each item:

   a. **Update status** to `in-progress` in run backlog. **The run backlog must be written to disk after every status change** — this is critical for resumability if the session is interrupted.

   b. **Execute the update:**
      - For update items: `ha_call_service("update", "install", entity_id=item.entity_id)`
      - For the MCP Server add-on: use the SSH bootstrap workaround (see below) instead of ha-mcp
      - For migration/task items: execute the described action (may involve ha-mcp calls, SSH commands, or config changes)

   c. **Wait for completion.** For HACS/add-on updates, the service call usually returns when done. For Core and Supervisor updates (which trigger restarts), **poll for availability** — call `ha_get_state("update.home_assistant_core_update")` every 30 seconds, up to a 6-minute max timeout. Do NOT use a fixed sleep duration.

   d. **Validate:**
      - `ha_check_config()` — must pass. If it fails, STOP. Do not continue.
      - Repairs count — compare to baseline. New repairs = log and flag.
      - Log scan — check for errors in the last 5 minutes. **Diff against the pre-update log baseline** to distinguish new errors from pre-existing ones.
      - **Integration state check** — `ha_get_integration()`, diff against baseline snapshot. Flag any integration that changed TO `not_loaded`/`setup_error`/`setup_retry`. If custom integration failed: restart once via `ha_restart(confirm=true)`, wait, recheck. If still failed, add to project backlog.
      - **Add-on verification** — For add-on updates, the update entity may show the old version even after success (entity lag). Use `ha_get_addon()` or SSH `ha apps info <slug>` as ground truth for version confirmation.
      - Conditional checks per heuristics (Sonos after Core, dashboard after UI components, etc.)
      - **Eval suites** — After Core or significant updates, match release note content against eval suite topics. Run relevant eval suites as smoke tests. Log results in run progress.

   e. **Log outcome** to run progress with timestamp.

   f. **Update run backlog** item status to `completed` or `failed` with notes.

   g. **If validation fails:** Stabilize the system (stop further updates, verify config is valid). Add the issue to the project backlog for human resolution. Do NOT continue to the next item.

   h. **After Core or Supervisor updates:** Call `ha_get_updates()` again. Compare against current run backlog. If new updates appeared, add them to the run backlog with `source: "re-inventory"`, assess risk and dependencies, and insert into execution order.

   i. **If release notes or validation surfaces non-update work** (deprecation migrations, entity renames, config changes): add to run backlog as `type: "migration"` or `type: "task"`, assess, execute or defer.

### Phase D: Closeout

10. **Git commit.** SSH to the Pi (using connection details from CLAUDE.md Configuration) and commit config changes. Use targeted `git add` with specific paths — NOT `git add -A`. Check `git status --short` for modified files and stage only those relevant to the updates.

11. **Refine heuristics.** Read the current heuristics file and the run progress. Update `Knowledge/update-heuristics.md`:
    - Update observation counts on existing patterns
    - Add new patterns discovered during this run
    - Consolidate redundant entries
    - Remove heuristics that haven't applied in 3+ consecutive runs
    - Do NOT append raw run history — distill into patterns only

12. **Write project progress entry.** Append a session summary to `progress.md` covering what was updated, issues encountered, heuristics refined, items deferred.

13. **Update project backlog.** Update relevant backlog items in `backlog.json` if appropriate.

14. **Clean up.** Delete the run-scoped temp directory.

15. **Report summary** to user.

## Error Handling

- **ha-mcp unreachable:** HA may be restarting. Poll every 30s up to 6 minutes. If still unreachable, stop and report.
- **Backup timeout:** The MCP call times out at 120s but the backup continues in the background. This is expected, not a failure. Verify completion via SSH `ha jobs info` as described in Step 8. Do NOT proceed with updates until the backup job shows `done: true` and the system state is `running`.
- **Backup freeze blocks operations:** If the system is in `freeze` state, OS updates, Supervisor restarts, and `ha backups thaw` (while a job is active) all fail. Wait for backup completion or defer the run. Do NOT attempt `ha host reboot` without user confirmation.
- **Config check fails:** STOP. Add to project backlog for human resolution.
- **Update entity stuck in progress:** Wait up to 5 minutes. Check via SSH `ha apps info` for ground truth.
- **Update entity shows old version after completion:** Known lag for add-ons. Verify via `ha_get_addon()` or SSH before concluding failure.
- **SSH connection fails:** Report. Git commit can be deferred as a manual follow-up.
- **Run interrupted:** Run-scoped files persist in temp. On re-invocation, offer to resume or start fresh.

## Autonomous Operation

This skill runs fully autonomously — no human checkpoints or approval prompts. When issues arise:

- **Transient failures** (integration not loading, service temporarily unavailable): retry once (restart HA if needed), then proceed if resolved. Log the event.
- **Blocking failures** (config check fails, update entity stuck, SSH unreachable): stabilize the system, add the issue to the project backlog for human resolution in a future session.
- **Ambiguous situations** (can't assess risk, conflicting heuristics, missing release notes): classify conservatively, proceed with extra validation, log the ambiguity for heuristic review at closeout.

## MCP Server Bootstrap Workaround

The MCP Server add-on cannot be updated via ha-mcp tools — updating it restarts the service that provides the tools. Detect this case (match the entity_id or add-on slug from CLAUDE.md Configuration > `addons.mcp_server_slug`) and route around it via SSH:

Resolve connection details and slug from CLAUDE.md Configuration, then:
```
ssh -i {ssh.key} {ssh.host} "ha apps update {addons.mcp_server_slug}"
```

Verify after via SSH: `ha apps info {slug} | grep version`

Then confirm MCP tools are back by calling `ha_get_updates()`.
