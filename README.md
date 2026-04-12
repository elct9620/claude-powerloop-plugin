# powerloop

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that enhances `/loop` with structured **Plan → Execute → Review → Sample** phases for quality-driven automation.

## Why powerloop?

The built-in `/loop` repeats a prompt on a schedule — useful, but blind. It doesn't track what's done, what failed, or whether the results are actually correct.

powerloop adds structure:

- **Progress tracking** per item across all phases
- **SubAgent dispatch** — the main session coordinates, SubAgents do the work
- **Automatic quality gates** — review scanning catches regressions, sampling builds confidence
- **Auto-stop** — no manual cleanup; the schedule removes itself when done

## Usage

```
/powerloop
```

The interactive setup guides you through six steps:

| Step | What you configure | Example |
|------|--------------------|---------|
| 1 | Goal | "Refactor all UI components" |
| 2 | Execute skills | `/coding:write` → `/coding:review` → `/coding:refactor` |
| 3 | Review skills | `/coding:review` → `/coding:refactor` |
| 4 | Interval | `5m` |
| 5 | Sample config | Enabled, 10 passes |
| 6 | Confirmation | Review summary, then approve |

After confirmation, powerloop creates a cron schedule and immediately begins the Plan phase.

## How it works

### Phase 1: Plan

Scans the codebase against your goal, builds a progress table with per-item status, and verifies every aspect of the goal maps to at least one item.

### Phase 2: Execute

Processes one item per cycle. Spawns a SubAgent (Sonnet) with your chosen execute skills. The SubAgent self-reviews before reporting back. New discoveries are appended as additional items.

### Phase 3: Review

Picks 2-3 items per cycle. Scanner SubAgents run in parallel to detect issues; a fixer SubAgent addresses failures sequentially. Fixed items are re-scanned in the next cycle to verify. Pauses if review cycles exceed 5.

### Phase 4: Sample

Randomly spot-checks 2-3 items per cycle. Consecutive clean passes increment a counter; any failure freezes it and triggers a fix. When the counter reaches the target, the schedule auto-stops.

> If sampling is disabled, the schedule auto-stops after the Review phase completes.

## Progress tracking

Progress is tracked in `.powerloop/YYYY-MM-DD-<name>.note.md` files. Users can `.gitignore .powerloop/` to exclude them, or commit them to preserve the full execution history.

Example mid-execution state:

```yaml
---
goal: "Refactor all UI components to use new Design System"
language: en
current_phase: review
started_at: 2026-04-04T10:00:00+08:00
interval: 5m
cron_id: a1b2c3d4
execute_skills: "/coding:write → /coding:review → /coding:refactor"
review_skills: "/coding:review → /coding:refactor"
sample_passes: 0/10
review_cycles: 1
---

| # | Item | Execute | Review | Sample | Notes |
|---|------|---------|--------|--------|-------|
| 1 | Extract Button component | done | done | pending | |
| 2 | Refactor Form component | done | in_progress | pending | Round 1 found issue |
| 3 | Unify Modal component API | done | pending | pending | |
| 4 | Replace legacy Card | done | pending | pending | |
| 5 | Update Sidebar navigation | done | pending | pending | Discovered during #2 |

## Log

| Cycle | Phase | Summary | Decision | Handoff |
|-------|-------|---------|----------|---------|
| 1 | plan | Decomposed into 4 items | Started from Button — deps first | |
| 6 | execute | Transitioned to review | | #3 #4 share overlay pattern — review together |
| 7 | review | #1 passed, #2 failed | Prioritized #2 fix — validation blocker | Nested form edge case untested — verify next cycle |
```

### Item statuses

| Status | Meaning |
|--------|---------|
| `pending` | Not started |
| `in_progress` | Currently being worked on |
| `done` | Completed |
| `failed` | Needs retry (skipped after 3 failures) |
| `skipped` | Intentionally skipped |

## Constraints

- **One batch per cycle** — execute processes 1 item, review/sample process 2-3 items, then stops until the next scheduled trigger
- **SubAgent-driven** — the main session dispatches only; all code changes happen in SubAgents
- **Failed items** retry automatically, skipped after 3 consecutive failures (tracked in Notes column)

## Cancellation

To stop a running powerloop, delete the cron schedule:

```
CronDelete <cron_id>
```

The `cron_id` is stored in the `.note.md` progress file and displayed when the loop starts.

## Requirements

- Claude Code with cron support enabled
- `/loop` feature available (same underlying `CronCreate` mechanism)

## License

[Apache-2.0](LICENSE)
