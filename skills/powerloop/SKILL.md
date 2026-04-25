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

Derive a short lowercase identifier from the goal for the progress file name (e.g., `refactor`, `spec-impl`, `migrate`).

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
- Allow disabling: if the user only wants Plan → Execute → Review (auto-stop after Review completes; set `sample_passes: 0/0` in frontmatter)

### Step 6: Language

Ask the user's preferred language for note content (progress table items, log entries, notes). Default: English. Preserve the user's original goal text in frontmatter without translation regardless of this setting.

### Step 7: Confirmation

Present a complete summary for user approval:

```
═══════════════════════════════════════════
  powerloop Configuration
═══════════════════════════════════════════

  Goal:          <goal>
  Progress file: .powerloop/<DATE>-<name>.note.md
  Interval:      <interval>

  ── Execute Phase ──────────────────────
  Skills:    <execute skills>
  Batch:     1 item per scheduled cycle
  Executor:  SubAgent (Sonnet) with self-review

  ── Review Phase ───────────────────────
  Skills:    <review skills>
  Batch:     2-3 items per scheduled cycle
  Scanner:   Two-stage blind — Haiku (broad scan) → Sonnet (verify aspects)
  Blinding:  Scanners see only Goal + target + criteria, no history or prior verdicts
  Fixer:     SubAgent (Sonnet) — receives findings, re-scanned blindly next cycle

  ── Sample Phase ──────────────────────
  Pass target: <N>/<target> clean spot-checks
  Batch:       2-3 random items per cycle
  Scanner:     Two-stage blind — Haiku (broad scan) → Sonnet (verify aspects)
  Blinding:    Same as Review — no signal that item already passed Review
  Rule:        Counter freezes on failure, resumes after fix
  On complete: Auto-stop schedule

  (If Sample disabled, auto-stop after Review completes)

═══════════════════════════════════════════
```

Wait for explicit confirmation before proceeding.

## Scheduling

After confirmation, compose the scheduled prompt, show it to the user for review, then create the cron job.

### Compose the Scheduled Prompt

Replace template variables (`<DATE>`, `<name>`, `<GOAL>`, `<EXECUTE_SKILLS>`, `<REVIEW_SKILLS>`, `<SAMPLE_TARGET>`) with the user's confirmed values. `<DATE>` is `YYYY-MM-DD` from `started_at`; `<name>` is the lowercase identifier. `<GOAL>` may be rephrased for clarity but must retain all qualifiers, constraints, and conditions from the user's confirmed goal — oversimplification causes SubAgent decisions to drift. Use this compact prompt template:

```
Read .powerloop/<DATE>-<name>.note.md for current phase, progress, and the latest Log Table entry's Handoff column for context from the previous cycle.

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
3. Mark done/failed based on SubAgent's self-review result, append any discovered items as new rows. On failure, increment `exec-fail:N` in Notes; at N=3 mark Execute=skipped AND Review=skipped (item is unimplementable, do not waste later phases on it)
4. When ALL Execute = done or skipped → set current_phase: review

### review
1. Pick 2-3 pending/failed Review items (skip items where Review = skipped, inherited from Execute = skipped)
2. Stage 1 — Broad scan (blind): spawn SubAgents (Haiku) in parallel. Each one receives ONLY: Goal, item name, target paths/scope, review_skills criteria. Do NOT pass Progress Table history, prior Execute/Review/Sample status, Log Table entries, other items' results, or any hint that the item was previously processed or fixed. Stage 1 should err conservative — when uncertain, return SUSPECT and let Stage 2 spend the deeper budget. Each scanner reports PASS or SUSPECT, where SUSPECT lists the *aspects* that raised concern (e.g., "error handling", "naming consistency") — not pre-baked verdicts or line-level findings.
3. Stage 2 — Verify (blind): for each SUSPECT item, spawn SubAgent (Sonnet) with ONLY: Goal, item name, target paths/scope, review_skills criteria, and the aspect labels from Stage 1 (e.g., "focus on: error handling, naming"). Do NOT forward Stage 1's specific findings, line numbers, quoted code, or reasoning — Stage 2 must inspect from scratch and reach its own conclusion. Report PASS or FAIL with specific issues.
4. Mark Review = done for items that passed Stage 1 directly (no SUSPECT) OR passed Stage 2 after a SUSPECT verdict
5. FAIL items → spawn fixer SubAgent (Sonnet) with full context (review_skills, Stage 2's specific issues) to fix — Fixer is NOT blind, it needs the findings. Mark Review = failed and increment `review-fail:N` in Notes; the next cycle re-scans these items blindly (do not signal to the next scanner that they were just fixed). Review failures have no skip threshold — items retry indefinitely. Persistent failure (e.g., review-fail ≥ 5) is a signal for human intervention, not for the loop to give up.
6. Track review_cycles in frontmatter
7. When ALL Review = done or skipped → if sample target > 0: set current_phase: sample, else: completed + CronDelete(cron_id)

### sample
1. If any items have Sample = failed, pick those first (up to the batch cap); fill remaining slots by randomly picking from items where Review = done and Sample = pending. Items where Review = skipped are excluded from the sample pool.
2. Stage 1 — Broad scan (blind): spawn SubAgents (Haiku). Same blinding rules and conservative bias as review Stage 1 — only Goal, item name, target paths, review_skills criteria. No history, no prior verdicts, no signal that the item passed Review.
3. Stage 2 — Verify (blind): for each SUSPECT item, spawn SubAgent (Sonnet) with only the aspect labels from Stage 1, not its specific findings. Report PASS or FAIL with specific issues.
4. Mark Sample = done for items that passed Stage 1 directly OR passed Stage 2. If every item this cycle passed → increment sample_passes
5. Any FAIL → counter does not increment this cycle, spawn fixer SubAgent (Sonnet) with full findings (Fixer is NOT blind), mark Sample = failed and increment `sample-fail:N` in Notes — the next cycle picks failed items first (per step 1) and re-scans them blindly. Sample failures have no skip threshold; persistent failure is a signal for human intervention.
6. When sample_passes reaches target → completed + CronDelete(cron_id)

## Blind Review Principle

Review and Sample phases are **adversarial quality gates**, not status confirmations. When a SubAgent sees that an item was already marked Execute=done, or that a previous reviewer said PASS, or that a Fixer just touched it, the cheapest path is to confirm the prior judgment instead of re-deriving it. That defeats the purpose of the gate.

The dispatcher (current session) is responsible for **stripping context before spawning Review/Sample SubAgents**. The SubAgent prompt must contain only what is needed to perform the check — Goal, target, criteria — and nothing about prior outcomes. Stage 2 receiving Stage 1's *aspect labels* (not findings) is a deliberate compromise: it focuses Sonnet's attention without anchoring its conclusion. Fixers are the only role exempt from blinding, because they need the specific issues to act on.

See `examples/blind-dispatch.md` for a side-by-side leaky-vs-blind prompt comparison and a pre-dispatch checklist.

## Constraints
- STOP after processing one batch — do NOT continue to the next item or cycle. Update the .note.md file and wait for the next scheduled trigger.
- Current session is dispatcher only — all work via SubAgents
- For review/sample SubAgents, strip context before dispatch: pass Goal + target + criteria only. Never include Progress Table rows, Log Table entries, prior status, or signals that an item was previously fixed. Fixers are exempt and receive full findings.
- One batch = execute: 1 item, review/sample: 2-3 items
- Failure counters are tracked separately per phase in the Notes column: `exec-fail:N`, `review-fail:N`, `sample-fail:N`. Only Execute has a skip threshold (3 consecutive fails → Execute=skipped, Review=skipped). Review and Sample retry indefinitely — powerloop is designed for long-running execution; getting stuck and signaling for human intervention is preferable to silently abandoning items.
- New discoveries: append rows with all statuses = pending
- After each batch, update the progress table: set processed items' status to done/failed and update current_phase in frontmatter if phase transition conditions are met
- After each batch, append a row to the Log Table (Cycle | Phase | Summary | Decision | Handoff)
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
Progress: .powerloop/<DATE>-<name>.note.md

Starting first cycle (Plan Phase) now...

Cancel: CronDelete <cron_id>
Auto-stop: after Sample Phase passes <target> times
```

Then immediately begin the Plan phase.

## Progress File Format

Track progress in `.powerloop/YYYY-MM-DD-<name>.note.md` where `YYYY-MM-DD` is the start date and `<name>` is the lowercase identifier derived from the goal (e.g., `.powerloop/2026-04-12-refactor.note.md`). Create the `.powerloop/` directory if it does not exist.

This directory-based approach lets users either `.gitignore .powerloop/` to exclude tracking files, or commit them to preserve the full execution history.

Refer to `examples/2026-04-04-refactor.note.md` for a complete mid-execution example.

### Frontmatter Fields

| Field | Description |
|-------|-------------|
| `goal` | The task objective |
| `language` | Language for note content (default: `en`) |
| `current_phase` | `plan` / `execute` / `review` / `sample` / `completed` |
| `started_at` | ISO 8601 timestamp |
| `interval` | Cron interval |
| `cron_id` | Schedule ID for CronDelete |
| `execute_skills` | Skills used in Execute phase |
| `review_skills` | Skills used in Review/Sample phases |
| `sample_passes` | `M/N` counter (current/target) |
| `review_cycles` | Number of completed review rounds |

### Progress Table

| # | Item | Execute | Review | Sample | Notes |
|---|------|---------|--------|--------|-------|

Status values: `pending`, `in_progress`, `done`, `failed` (retry next cycle), `skipped` (reason in Notes).

### Log Table

Append one row per cycle. Each entry serves as a handoff to the next cron trigger — a fresh conversation that reads this file as its only context.

| Cycle | Phase | Summary | Decision | Handoff |
|-------|-------|---------|----------|---------|

- **Summary** — what happened this cycle, brief factual record
- **Decision** — judgment calls and their reasoning; leave empty for routine cycles
- **Handoff** — what the next cycle needs to know: caveats, unfinished context, suggested approach

Write progress table items, log entries, and notes in the language specified by the `language` frontmatter field. Preserve the user's original goal text in frontmatter without translation.
