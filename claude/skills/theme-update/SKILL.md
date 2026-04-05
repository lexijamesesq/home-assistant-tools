---
name: theme-update
description: >
  Triggers when the user says "update the theme", "merge theme update",
  "/theme-update", or similar requests to merge a new Catppuccin distribution
  release into the Catppuccin Mush theme. Also trigger when HACS updates
  the Catppuccin distribution and the user wants to reconcile.
user_invokable: true
allowed-tools:
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - Bash(ssh:*)
  - Bash(scp:*)
  - Bash(python3:*)
  - Skill
  - mcp__ha-mcp__ha_call_service
  - mcp__ha-mcp__ha_check_config
  - mcp__claude-in-chrome__*
---

# /theme-update — Catppuccin Mush Theme Merge

Merge a new Catppuccin distribution release into the Catppuccin Mush theme, preserving Mush overrides and applying the required HA 2026.x compatibility fixes.

## Invocation

```
/theme-update
```

Run after HACS updates the Catppuccin distribution in `/config/themes/catppuccin/`.

## Prerequisites

- SSH access to HA Pi (connection details in CLAUDE.md Configuration > `ssh.*`)
- Chrome MCP for visual validation
- Read `Knowledge/ha-theme-architecture.md` before proceeding

## Instructions

### Step 1: Capture Visual Baseline

Invoke `/visual-diff before theme-update-{version}` where `{version}` is the new distribution version.

### Step 2: Extract New Distribution Frappe Section

SSH to the Pi (using connection details from CLAUDE.md Configuration) and extract the `Catppuccin Frappe:` section from `/config/themes/catppuccin/catppuccin.yaml`. The Frappe section starts with `Catppuccin Frappe:` and ends before the next flavor (`Catppuccin Latte:`, `Catppuccin Macchiato:`, or `Catppuccin Mocha:`).

Save locally to `/tmp/catppuccin_frappe_new.yaml`.

### Step 3: Read Current Mush Theme

Read `/config/themes/catppuccin_mush/catppuccin_mush.yaml` from the Pi. Save locally to `/tmp/catppuccin_mush_current.yaml`.

### Step 4: Diff Distribution Changes

Compare the new Frappe section against the distribution base embedded in the current Mush theme. Identify:

- **Palette changes** — any new/modified `catppuccin-*` hex values (rare, but check)
- **New ramp values** — additional `ha-color-*-XX` stops or new color groups
- **New semantic variables** — additional `ha-color-*` semantic mappings (HA releases add these)
- **New standard variables** — additional HA CSS variables
- **Structural changes** — new sections, removed variables

Report the diff summary to the user before proceeding.

### Step 5: Apply Ramp Flip

The distribution computes ramps with inverted direction (05=lightest, 95=darkest). HA 2026.x dark-mode components expect 05=darkest, 95=lightest.

**Flip all ramp sections** by swapping values at mirrored stops:

| Stop | Swap With |
|------|-----------|
| 05 | 95 |
| 10 | 90 |
| 20 | 80 |
| 30 | 70 |
| 40 | 60 |
| 50 | (unchanged) |

Apply to ALL ramp groups: primary, neutral, orange, red, green, and any new groups added by the distribution.

### Step 6: Mirror Semantic Stop References

Update ALL semantic color references that point to ramp stops. Apply the same mirroring:

- `primary-95` → `primary-05`, `primary-90` → `primary-10`, etc.
- `neutral-95` → `neutral-05`, `neutral-90` → `neutral-10`, etc.
- Same for orange, red, green, and any new groups.

**Do NOT change** references to:
- Palette colors (`catppuccin-text`, `catppuccin-blue`, etc.)
- `--white-color`, `--primary-color`, or other non-ramp references
- The `50` stop (midpoint, unchanged)

### Step 7: Merge Mush Overrides

The current Mush theme has overrides marked with `# Mush:` comments. These are aesthetic choices that must be preserved.

**Merge strategy for each section:**

1. **Palette** (`catppuccin-*` vars): Take new distribution values. These are upstream truth.

2. **Ramps** (`ha-color-*-XX`): Take new distribution values (already flipped in Step 5).

3. **Semantics** (`ha-color-*` non-ramp): Take new distribution values (already mirrored in Step 6), EXCEPT for lines marked `# Mush:` — preserve those.

4. **Standard HA variables**: For each variable:
   - If the current Mush theme has a `# Mush:` override → keep the Mush value
   - If no Mush override → take the new distribution value
   - If the distribution adds a NEW variable not in current Mush → add it

5. **Mush-only sections** (sidebar, cards, switches, icons, badges, states, tables, dialogs, selects, Mushroom vars): Preserve entirely. These don't exist in the distribution.

### Step 8: Post-Processing

Apply these mandatory fixes:

1. **Remove `modes:` declaration** — if the distribution adds `modes: dark: {}`, delete it. HA 2026.x injects 523 hardcoded dark-mode CSS vars as inline styles on `hui-view-container` when this is present. Add the explanatory comment from the current theme.

2. **Ensure background variables** — verify these exist:
   ```yaml
   lovelace-background: var(--primary-background-color)
   view-background: var(--primary-background-color)
   ```

3. **Ensure `ha-card-background`** — verify this exists:
   ```yaml
   ha-card-background: var(--card-background-color)
   ```

4. **Theme name** — must be `Catppuccin Mush:` (not the distribution's `Catppuccin Frappe:`).

5. **File header** — update the version reference in the header comment to the new distribution version.

### Step 9: Validate

1. Validate YAML locally: `python3 -c "import yaml; yaml.safe_load(open('/tmp/catppuccin_mush.yaml'))"`
2. Check for duplicate keys (HA YAML takes first value — duplicates silently break)
3. Verify no `modes:` declaration exists
4. Verify ramp direction: `ha-color-neutral-05` should have a LOWER RGB sum than `ha-color-neutral-95`

### Step 10: Deploy and Visual Diff

1. SCP the new theme to Pi using SSH connection details from CLAUDE.md Configuration (`ssh.key`, `ssh.host`, `ssh.config_path`)
2. Reload themes: `frontend.reload_themes`
3. Set theme: `frontend.set_theme` with name `Catppuccin Mush` (no mode parameter)
4. Invoke `/visual-diff after theme-update-{version}`
5. Present before/after comparison to user

### Step 11: Finalize

If user approves:
1. Commit on Pi using SSH connection details from CLAUDE.md Configuration
2. Update the version reference in `Knowledge/ha-theme-architecture.md` if the architecture changed

If user rejects:
1. Revert: `git checkout -- themes/catppuccin_mush/` on Pi
2. Reload themes and re-set theme
3. Investigate what went wrong

## Why the Ramp Flip Is Needed

The Catppuccin Tera template computes ramps by mixing accent colors toward text (lighter) and base (darker). For dark flavors like Frappe, this produces 05=lightest, 95=darkest.

HA 2026.x dark-mode components (`hui-view-container`, `hui-sections-view`) apply their own CSS that remaps ramp stops using dark-mode semantics — e.g., `surface-default = neutral-10`. With the inverted ramp, `neutral-10` resolves to a LIGHT value, producing light backgrounds on a dark theme.

Flipping the ramp makes `neutral-10` resolve to a DARK value, which is what HA's dark-mode components expect.

This is a distribution-level bug that the Catppuccin maintainer considers intentional ("use light-mode semantics on an inverted palette"). Our flip is the workaround.

## Why modes: Must Be Removed

When `modes: dark: {}` is present, HA's frontend JavaScript injects 523 CSS custom properties as inline styles on `hui-view-container`. These include hardcoded values like `--primary-text-color: #e1e1e1` that override ALL inherited theme values. Removing the declaration prevents this injection.
