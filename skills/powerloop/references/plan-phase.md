# Plan Phase Template

## Purpose

Analyze the goal, scan the codebase, and produce a progress tracking table in `<name>.local.md`.

## Prompt Template

```
## Plan Phase

Read `<NAME>.local.md`. If it does not exist, this is the first execution — create it now.

### First Execution

1. Analyze the goal: <GOAL>
2. Scan the codebase to identify all items that need work
3. Create `<NAME>.local.md` with the progress table below
4. Validate: review the table against the goal — ensure completeness

### Subsequent Execution (items added dynamically)

1. Read `<NAME>.local.md`
2. If current phase is still `plan`, review the table for completeness
3. Mark plan phase as `done` when satisfied, set current phase to `execute`

### Progress Table Format

Write `<NAME>.local.md` with this structure:

---
goal: "<GOAL>"
current_phase: plan
started_at: <ISO_8601_TIMESTAMP>
interval: <INTERVAL>
cron_id: <CRON_ID>
execute_skills: "<EXECUTE_SKILLS>"
review_skills: "<REVIEW_SKILLS>"
sample_passes: 0/<SAMPLE_TARGET>
---

| # | Item | Execute | Review | Sample | Notes |
|---|------|---------|--------|--------|-------|
| 1 | <item description> | pending | pending | pending | |
| 2 | <item description> | pending | pending | pending | |

### Status Values

- `pending` — not started
- `in_progress` — currently being worked on
- `done` — completed
- `failed` — needs retry
- `skipped` — intentionally skipped with reason in Notes

### Dynamic Item Discovery

During any phase, if new items are discovered (e.g., a refactor reveals additional work):
1. Append new rows to the table
2. Set all phase statuses to `pending` for new items
3. Add a note: "Discovered during <phase> of item #N"
```

## Key Rules

- The progress file MUST use `.local.md` extension to avoid being committed
- Place the file in the project root directory
- The frontmatter tracks global state; the table tracks per-item state
- All items must reach `done` in Execute before moving to Review phase
- All items must reach `done` in Review before moving to Sample phase
