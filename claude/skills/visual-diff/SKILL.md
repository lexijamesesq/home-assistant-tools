---
name: visual-diff
description: >
  Triggers when the user says "visual diff", "capture screenshots",
  "before/after comparison", "/visual-diff", or similar requests to
  capture dashboard screenshots for visual validation of UI changes.
  Use this skill whenever dashboard appearance changes need visual
  verification — theme updates, card modifications, layout changes.
argument-hint: "<phase> <label> [paths...]"
user_invokable: true
allowed-tools:
  - Read
  - Write
  - Bash(mkdir:*)
  - mcp__claude-in-chrome__*
---

# /visual-diff — Dashboard Visual Comparison

Capture before/after screenshots of dashboard views to visually validate UI changes.

## Invocation

```
/visual-diff before theme-update-2.1.4
/visual-diff after theme-update-2.1.4
/visual-diff compare theme-update-2.1.4
```

## Arguments

Format: `<phase> <label> [paths...]`

- **phase**: `before`, `after`, or `compare`
- **label**: descriptive name for the change (used in filenames)
- **paths**: optional list of dashboard view paths to capture. Defaults to the standard set below.

## Default Dashboard Views

If no paths specified, use the dashboard name and views list from CLAUDE.md Configuration > `dashboard.name` and `dashboard.views`. Construct paths as `/{dashboard.name}/{view}` for each view in the list.

## Instructions

### Phase: `before`

1. Parse the label and optional paths from arguments
2. Create the output directory: `Screenshots/{label}/before/`
3. For each dashboard path:
   a. Navigate to the URL using Chrome MCP (construct from CLAUDE.md Configuration > `ha.local_url` + path)
   b. Wait for the view to fully render (check for card content in DOM)
   c. Take a screenshot using Chrome MCP gif_creator (single frame) or computer tool
   d. Save as `Screenshots/{label}/before/{view-name}.png` where view-name is the last path segment
4. Report: "Captured N before screenshots for {label}"

### Phase: `after`

1. Parse the label and optional paths from arguments
2. Create the output directory: `Screenshots/{label}/after/`
3. For each dashboard path:
   a. Navigate, wait for render, capture screenshot
   b. Save as `Screenshots/{label}/after/{view-name}.png`
4. Report the before/after pairs to the user for visual comparison
5. Read each before/after image pair and present them sequentially

### Phase: `compare`

1. Read existing before/after screenshots from `Screenshots/{label}/`
2. Present each pair to the user for review
3. Ask: "Do the changes look correct?"

## Output Location

All screenshots saved under the project root:
```
Screenshots/
  {label}/
    before/
      home.png
      lights.png
      ...
    after/
      home.png
      lights.png
      ...
```

## Notes

- This skill is a reusable utility. Any workflow that modifies dashboard appearance can invoke it.
- The Chrome MCP tab used should be from the current session's tab group.
- If a view fails to load, note it and continue with remaining views.
- Screenshots are project artifacts, not committed to git.
