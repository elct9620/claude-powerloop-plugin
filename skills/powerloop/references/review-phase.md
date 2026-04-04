# Review Phase Template

## Purpose

Systematically review every item for quality. Scan in parallel, fix issues sequentially. Review phase may repeat multiple times until all items pass.

## Prompt Template

```
## Review Phase

Read `<NAME>.local.md`. Current phase must be `review`.

### Per Execution (2-3 items per cron cycle)

1. Read the progress table
2. Select next 2-3 items with Review status `pending` or `failed` (lowest #)
3. **Parallel Scan**: Spawn SubAgents (Sonnet or Haiku) to review each item concurrently:

   Scanner SubAgent instructions:
   - Goal: <GOAL>
   - Item: <item description>
   - Skills to use: <REVIEW_SKILLS>
   - Check for: correctness, code quality, consistency with goal
   - Report: PASS or FAIL with specific issues found

4. **Sequential Fix**: For items that FAIL, spawn a SubAgent (Sonnet) to fix:

   Fixer SubAgent instructions:
   - Goal: <GOAL>
   - Item: <item description>
   - Issues found: <scanner report>
   - Skills to use: <REVIEW_SKILLS>
   - Fix the reported issues
   - Commit changes if applicable

5. After all items processed:
   - PASS → set Review status to `done`
   - FAIL + fixed → set Review status to `done`
   - FAIL + not fixable → set Review status to `failed`, add details to Notes

### Phase Transition

After updating the table, check:
- If ALL items have Review status `done`:
  - If Sample is enabled (sample_passes target > 0) → set `current_phase: sample`
  - If Sample is disabled (sample_passes target = 0) → set `current_phase: completed`, call `CronDelete` with `cron_id` to stop the schedule, and output a completion summary
- Otherwise → next cron cycle picks up remaining items

### Review Cycle Limit

- Track review cycles in frontmatter: `review_cycles: N`
- If review cycles exceed 5 without all items passing, pause and notify:
  "Review phase has exceeded 5 cycles. Manual intervention may be needed."

### SubAgent Guidelines

- Scanner: use Sonnet or Haiku for parallel scanning
- Fixer: use Sonnet for sequential fixes
- Current session dispatches — do not perform review/fix directly
```

## Key Rules

- Process 2-3 items per cron cycle
- Scan in parallel (Sonnet/Haiku), fix sequentially (Sonnet)
- Current session is dispatcher only
- All items must reach `done` before transitioning to Sample
- Maximum 5 review cycles before pausing for human intervention
