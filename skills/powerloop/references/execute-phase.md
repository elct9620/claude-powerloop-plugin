# Execute Phase Template

## Purpose

Work through each item in the progress table using the specified skills/commands. Focus on completing the broad strokes — details are refined in Review.

## Prompt Template

```
## Execute Phase

Read `<NAME>.local.md`. Current phase must be `execute`.

### Per Execution (1 item per cron cycle)

1. Read the progress table
2. Find the next item with Execute status `pending` (lowest #)
3. Set its Execute status to `in_progress`
4. Spawn a SubAgent (Sonnet) to handle the item:

   SubAgent instructions:
   - Goal: <GOAL>
   - Item: <item description>
   - Skills to use: <EXECUTE_SKILLS>
   - Focus on completing the main work, not perfection
   - Report what was done and any new items discovered

5. After SubAgent completes:
   - Set Execute status to `done` (or `failed` if issues)
   - If new items were discovered, append them to the table
   - Update Notes with a brief summary

### Phase Transition

After updating the table, check:
- If ALL items have Execute status `done` → set `current_phase: review`
- Otherwise → next cron cycle picks up the next pending item

### Error Handling

- If a SubAgent fails, set Execute status to `failed` and add error details to Notes
- Failed items will be retried on the next cycle
- If an item fails 3 times, mark as `skipped` with reason and continue

### SubAgent Guidelines

- Use model: Sonnet for execution
- The current session acts as dispatcher — do not execute work directly
- Each SubAgent receives only the context for its specific item
- SubAgent should commit changes if applicable
```

## Key Rules

- Process exactly 1 item per cron cycle
- Use SubAgent (Sonnet) for actual work, current session dispatches
- All items must reach `done` before transitioning to Review
- Failed items get retried up to 3 times
- New discoveries are appended with all statuses set to `pending`
