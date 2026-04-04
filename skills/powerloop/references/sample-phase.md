# Sample Phase Template

## Purpose

Spot-check random items to build confidence in overall quality. Uses a passed counter that only increments on clean runs and freezes on issues.

## Prompt Template

```
## Sample Phase

Read `<NAME>.local.md`. Current phase must be `sample`.

### Per Execution (2-3 random items per cron cycle)

1. Read the progress table and frontmatter `sample_passes: M/<SAMPLE_TARGET>`
2. Randomly select 2-3 items (use different items than last cycle when possible)
3. **Parallel Scan**: Spawn SubAgents (Sonnet or Haiku) to review each item:

   Scanner SubAgent instructions:
   - Goal: <GOAL>
   - Item: <item description>
   - Perform a thorough quality check
   - Report: PASS or FAIL with specific issues

4. Evaluate results:

   **All items PASS** (clean run):
   - Increment passed counter: `sample_passes: (M+1)/<SAMPLE_TARGET>`
   - Record which items were checked in Notes

   **Any item FAILS** (dirty run):
   - Do NOT increment counter (freeze at current M)
   - Spawn SubAgent (Sonnet) to fix the issue
   - Set the failed item's Sample status to `failed`
   - After fix, set Sample status back to `pending`
   - Record the failure in Notes

### Completion

After updating the counter, check:
- If `sample_passes` reaches `<SAMPLE_TARGET>/<SAMPLE_TARGET>`:
  1. Set `current_phase: completed`
  2. Call `CronDelete` with `cron_id` from frontmatter to stop the schedule
  3. Output a summary:
     "powerloop completed: <GOAL>
      Items processed: N
      Review cycles: X
      Sample passes: <SAMPLE_TARGET>/<SAMPLE_TARGET>
      Duration: <elapsed time>"

### Counter Rules

- Counter starts at 0/<SAMPLE_TARGET>
- Only increments when ALL sampled items pass in a single cycle
- Freezes (stays at current value) when any issue is found
- Never decrements
- Issues found during sampling trigger immediate fix, not counter reset

### SubAgent Guidelines

- Scanner: use Sonnet or Haiku for parallel spot-checks
- Fixer: use Sonnet for issue remediation
- Current session dispatches — do not perform checks directly
- Randomize item selection to maximize coverage
```

## Key Rules

- Randomly sample 2-3 items per cycle (avoid repeating same items)
- Passed counter only increments on fully clean runs
- Counter freezes (not resets) when issues are found
- Auto-stop via CronDelete when target reached
- Default target: 10 passes, user-configurable
