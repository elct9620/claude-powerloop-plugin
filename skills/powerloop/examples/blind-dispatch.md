# Blind Dispatch Example — Review/Sample Phase

This example shows what `Review` and `Sample` SubAgent prompts should look like when the dispatcher correctly strips context. It contrasts a leaky prompt (anti-pattern) with a blind prompt (correct).

## Scenario

Progress note has a Notes column reading: `fail:1 — fixer applied patch last cycle for unhandled null on query prop at line 42`. Item under review: `src/components/SearchBar.tsx`. Goal: refactor UI components to use new Button/Input primitives, preserving prop APIs.

## ❌ Anti-pattern — Leaky Stage 1 prompt

```
Target file: src/components/SearchBar.tsx

Important context: This file was previously marked Review = failed.
In cycle 4, Stage 2 found an unhandled null on the `query` prop at
line 42, and a fixer SubAgent applied a patch. You are re-scanning
to verify the fix held.

Pay special attention to null handling on `query` around line 42.
```

Why this is wrong: the scanner is told *where* to look and *what* the previous verdict was. The cheapest path is to confirm the fix at line 42 and PASS — every other latent bug in the file is now invisible. The gate has been bypassed by its own dispatcher.

## ✅ Correct — Blind Stage 1 prompt

```
You are reviewing a single file against a refactor goal. Report only
PASS or SUSPECT. If SUSPECT, list the *aspects* that concern you
(short labels like "error handling", "naming consistency", "prop API
drift") — do NOT include line numbers, quoted code, or pre-baked
verdicts.

Goal: Refactor all UI components in src/components/ to use the new
Button and Input primitives, preserving existing prop APIs.

Target file: src/components/SearchBar.tsx

Review criteria (from /coding:review and /coding:testing):
- Component uses the new Button and Input primitives where appropriate
- Existing prop API surface is preserved
- Style/architectural alignment with the rest of the codebase
- Tests still pass and cover the refactored behavior

Output:
PASS
or
SUSPECT: <aspect-label-1>, <aspect-label-2>, ...
```

The scanner forms its own first impression. If the fix didn't hold, it surfaces naturally; if a different bug exists, it is just as likely to be flagged.

## Stage 2 — aspect labels only

When Stage 1 returns `SUSPECT: error handling, naming`, Stage 2 receives:

```
Focus your inspection on these aspects: error handling, naming.
```

Not: `Stage 1 found unhandled null on query prop at line 42`. Stage 2 must re-derive the specific finding, otherwise it just rubber-stamps Stage 1.

## Fixer — NOT blind

Once Stage 2 returns FAIL with a specific finding, the Fixer prompt **does** receive the full context — file path, line numbers, quoted issue, and any prior-cycle constraints needed to avoid regression (e.g., "preserve existing prop APIs — do not rename"). Fixers act on findings; blinding them would force them to re-inspect and slow the loop.

## Dispatcher checklist

Before sending any Review/Sample SubAgent prompt, scan it for these strings and remove them:

- `Review = done|failed|pending`, `Execute = done`, `sample_passes`
- "previously", "last cycle", "cycle N", "prior", "recheck"
- "fixer", "patched", "fixed by", "just fixed"
- "Stage 1 said", "Stage 2 said", "PASS", "FAIL", "SUSPECT" (when referring to past verdicts)
- specific line numbers or code excerpts from the Notes/Log columns

If any survives, the prompt is leaky. Strip it before dispatch.
