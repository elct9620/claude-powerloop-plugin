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

Rephrasing for clarity is fine, but do not drop qualifiers, scope constraints, or edge-case notes — these details affect how SubAgents make judgment calls later. If you need to condense, confirm the rewritten goal with the user before proceeding.

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
  Executor:  SubAgent (Sonnet) with self-review

  ── Review Phase ───────────────────────
  Skills:    <review skills>
  Batch:     2-3 items per scheduled cycle
  Scanner:   SubAgent (Sonnet/Haiku) in parallel
  Fixer:     SubAgent (Sonnet) — fixes applied, re-scanned next cycle
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

After confirmation, compose the scheduled prompt, show it to the user for review, then create the cron job.

### Compose the Scheduled Prompt

Replace template variables (`<GOAL>`, `<NAME>`, `<EXECUTE_SKILLS>`, `<REVIEW_SKILLS>`, `<SAMPLE_TARGET>`) with the user's confirmed values. `<GOAL>` may be rephrased for clarity but must retain all qualifiers, constraints, and conditions from the user's confirmed goal — oversimplification causes SubAgent decisions to drift. Use this compact prompt template:

```
Read <NAME>.local.md for current phase and progress.

Goal: <GOAL>
Execute skills: <EXECUTE_SKILLS>
Review skills: <REVIEW_SKILLS>
Sample rule: 0/<SAMPLE_TARGET> — increment on clean run, freeze on failure

## Phase Rules

### plan
1. If execute_skills are provided, invoke them via Skill tool — let their workflow and principles inform how to decompose the goal
2. Scan the codebase against the goal
3. Build or refine the progress table (| # | Item | Execute | Review | Sample | Notes |) — if skills were invoked, align item granularity and ordering with the skill's workflow
4. Completeness check: every aspect of the goal must map to at least one item; if gaps remain, keep refining
5. When table covers the full goal, set current_phase: execute

### execute
1. Pick next pending Execute item (lowest #), set to in_progress
2. Spawn SubAgent (Sonnet) with the goal context and execute_skills — SubAgent must self-review its work before reporting back
3. Mark done/failed based on SubAgent's self-review result, append any discovered items as new rows
4. When ALL Execute = done → set current_phase: review

### review
1. Pick 2-3 pending/failed Review items
2. Spawn scanner SubAgents (Sonnet/Haiku) in parallel → each must report PASS or FAIL with specific issues
3. PASS items → mark Review = done
4. FAIL items → spawn fixer SubAgent (Sonnet) with review_skills to fix the reported issues, then mark Review = failed
5. Track review_cycles in frontmatter; pause if > 5 — failed items are re-scanned in the next cycle to verify fixes
6. When ALL Review = done → if sample target > 0: set current_phase: sample, else: completed + CronDelete(cron_id)

### sample
1. Randomly pick 2-3 items, spawn scanner SubAgents (Sonnet/Haiku) → each must report PASS or FAIL with specific issues
2. All PASS → increment sample_passes
3. Any FAIL → freeze counter, spawn fixer SubAgent (Sonnet) to fix the reported issues, then mark Sample = failed — fixed items are re-scanned in the next cycle to verify
4. When sample_passes reaches target → completed + CronDelete(cron_id)

## Constraints
- STOP after processing one batch — do NOT continue to the next item or cycle. Update the .local.md file and wait for the next scheduled trigger.
- Current session is dispatcher only — all work via SubAgents
- One batch = execute: 1 item, review/sample: 2-3 items
- Failed items retry next cycle, skip after 3 failures
- New discoveries: append rows with all statuses = pending
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

### Preview Prompt

Display the composed prompt in a fenced code block and ask the user to confirm before scheduling. Only proceed to CronCreate after explicit approval.

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

## Progress File Format

Track progress in `<NAME>.local.md` at project root. The `.local.md` extension prevents accidental commits in target projects.

Refer to `examples/REFACTOR.local.md` for a complete mid-execution example.

### Frontmatter Fields

| Field | Description |
|-------|-------------|
| `goal` | The task objective |
| `current_phase` | `plan` / `execute` / `review` / `sample` / `completed` |
| `started_at` | ISO 8601 timestamp |
| `interval` | Cron interval |
| `cron_id` | Schedule ID for CronDelete |
| `execute_skills` | Skills used in Execute phase |
| `review_skills` | Skills used in Review/Sample phases |
| `sample_passes` | `M/N` counter (current/target) |
| `review_cycles` | Number of completed review rounds |

### Status Values

- `pending` — not started
- `in_progress` — currently being worked on
- `done` — completed
- `failed` — needs retry
- `skipped` — intentionally skipped (reason in Notes)
