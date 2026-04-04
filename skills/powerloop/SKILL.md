---
name: powerloop
description: "This skill should be used when the user runs /powerloop, asks to set up an automated task loop, wants iterative refactoring with quality checks, or needs to automate multi-step feature implementation across many files. It creates a cron-based schedule with Plan, Execute, Review, and Sample phases that track progress per item, dispatch work to SubAgents, and auto-stop when quality sampling passes."
argument-hint: "[goal, e.g. 'refactor all UI components']"
allowed-tools: ["Read", "Write", "Edit", "Glob", "Grep", "Bash", "Agent", "CronCreate", "CronDelete", "CronList", "Skill", "AskUserQuestion"]
---

# powerloop — Structured Loop with Quality Phases

An enhanced `/loop` that wraps recurring tasks in a **Plan → Execute → Review → Sample** cycle. Track progress per item, dispatch work to SubAgents, and auto-stop when quality sampling passes.

## Interactive Setup Flow

When invoked, guide the user through these questions in order. Adapt phrasing naturally — do not dump all questions at once.

### Step 1: Goal

Ask what the user wants to accomplish. Examples:
- "Refactor all UI components"
- "Implement remaining features from SPEC.md"
- "Convert all class components to function components"

Derive a short identifier from the goal for the progress file name (e.g., `REFACTOR`, `SPEC-IMPL`, `MIGRATE`).

### Step 2: Execution Approach

Ask what skills or commands to use during the Execute phase. Examples:
- `/coding:write` → `/coding:review` → `/coding:refactor`
- `/coding:fix` for each item
- Custom instructions

### Step 3: Review Approach

Ask what skills or commands to use during Review and Sample phases. Examples:
- `/coding:review` → `/coding:refactor`
- `/coding:testing` to verify tests pass
- Custom review criteria

### Step 4: Interval

Ask the preferred interval between executions. Suggest `5m` as default. Support the same format as `/loop`: `Ns`, `Nm`, `Nh`, `Nd`.

### Step 5: Sample Configuration

Ask whether to enable the Sample phase and the pass target:
- Default: enabled, 10 passes required
- Allow customization: "How many consecutive passes required? (default: 10)"
- Allow disabling: if the user only wants Plan → Execute → Review (auto-stop after Review completes)

### Step 6: Confirmation

Present a complete summary for user approval:

```
═══════════════════════════════════════════
  powerloop Configuration
═══════════════════════════════════════════

  Goal:          <goal>
  Progress file: <NAME>.local.md
  Interval:      <interval>

  ── Execute Phase ──────────────────────
  Skills:    <execute skills>
  Batch:     1 item per scheduled cycle
  Executor:  SubAgent (Sonnet)

  ── Review Phase ───────────────────────
  Skills:    <review skills>
  Batch:     2-3 items per scheduled cycle
  Scanner:   SubAgent (Sonnet/Haiku) in parallel
  Fixer:     SubAgent (Sonnet) sequential
  Max cycles: 5 review cycles before pause

  ── Sample Phase ──────────────────────
  Pass target: <N>/<target> clean spot-checks
  Batch:       2-3 random items per cycle
  Rule:        Counter freezes on failure, resumes after fix
  On complete: Auto-stop schedule

  (If Sample disabled, auto-stop after Review completes)

═══════════════════════════════════════════
```

Wait for explicit confirmation before proceeding.

## Scheduling

After confirmation, compose the scheduled prompt and create the cron job.

### Compose the Scheduled Prompt

Read all four reference files to build the complete prompt:
- `references/plan-phase.md`
- `references/execute-phase.md`
- `references/review-phase.md`
- `references/sample-phase.md`

Replace all template variables (`<GOAL>`, `<NAME>`, `<INTERVAL>`, `<EXECUTE_SKILLS>`, `<REVIEW_SKILLS>`, `<SAMPLE_TARGET>`, `<CRON_ID>`) with the user's confirmed values.

Combine into a single scheduled prompt with this structure:

```
Read <NAME>.local.md to determine the current phase and progress.
Based on current_phase, follow the matching phase instructions below.

[Plan Phase rules]
[Execute Phase rules]
[Review Phase rules]
[Sample Phase rules]

Goal: <goal>
Execute skills: <execute skills>
Review skills: <review skills>
Sample target: <N> (0 = disabled, auto-stop after Review)
```

### Interval to Cron Conversion

Follow the same rules as `/loop`:

| Pattern | Cron Expression |
|---------|----------------|
| `Nm` (N ≤ 59) | `*/N * * * *` |
| `Nm` (N ≥ 60) | `0 */H * * *` |
| `Nh` | `0 */N * * *` |
| `Nd` | `0 0 */N * *` |
| `Ns` | round up to `ceil(N/60)m` |

### Create the Schedule

1. Call `CronCreate` with:
   - `cron`: the computed expression
   - `prompt`: the composed prompt
   - `recurring`: `true`
2. Note the returned `cron_id`
3. Immediately execute the first cycle (Plan phase) — do not wait for first cron fire

### Confirm to User

```
powerloop started!

Schedule ID: <cron_id>
Cron: <cron expression> (every <interval>)
Progress: <NAME>.local.md

Starting first cycle (Plan Phase) now...

Cancel: CronDelete <cron_id>
Auto-stop: after Sample Phase passes <target> times
Expiry: auto-expires after 7 days if not completed
```

Then immediately begin the Plan phase.

## Phase Reference Files

Detailed rules for each phase are in `references/`:

- **`references/plan-phase.md`** — Progress table creation, status values, dynamic item discovery
- **`references/execute-phase.md`** — SubAgent dispatch, 1 item per cycle, error handling
- **`references/review-phase.md`** — Parallel scan + sequential fix, 2-3 items per cycle, cycle limits
- **`references/sample-phase.md`** — Random sampling, passed counter rules, auto-stop via CronDelete

## Example

A sample progress file mid-execution is available at `examples/REFACTOR.local.md`. Refer to it for the expected format and status patterns.

## Progress File Convention

- File name: `<NAME>.local.md` in project root (e.g., `REFACTOR.local.md`)
- The `.local.md` extension ensures it is not committed to version control
- Frontmatter contains global state (phase, counters, config)
- Table body contains per-item tracking across all phases
