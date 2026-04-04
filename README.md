# powerloop

Enhanced `/loop` for Claude Code with structured **Plan → Execute → Review → Sample** phases.

## What it does

Instead of blindly repeating a prompt, powerloop:

1. **Plan** — Scans the codebase and builds a progress tracking table
2. **Execute** — Works through items one at a time using SubAgents
3. **Review** — Parallel quality scanning + sequential fixing
4. **Sample** — Random spot-checks with a passed counter that auto-stops when confident

## Usage

```
/powerloop
```

Interactive setup guides you through:
- Goal definition
- Execute/Review skill selection
- Interval and sample configuration

## Progress Tracking

Progress is tracked in `<NAME>.local.md` files (git-ignored) with per-item status across all phases.

## Requirements

- Claude Code with cron support enabled
- `/loop` feature available (same underlying CronCreate mechanism)
