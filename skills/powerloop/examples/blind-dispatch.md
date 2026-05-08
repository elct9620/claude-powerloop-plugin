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
You are reviewing a single file against a refactor goal.

Before inspecting, invoke the following skills via the Skill tool to
load the review criteria — these define what to check for and how:
  - /coding:review
  - /coding:testing

Then report only PASS or SUSPECT. If SUSPECT, list the *aspects* that
concern you (short labels like "error handling", "naming consistency",
"prop API drift") — do NOT include line numbers, quoted code, or
pre-baked verdicts.

Goal: Refactor all UI components in src/components/ to use the new
Button and Input primitives, preserving existing prop APIs.

Target file: src/components/SearchBar.tsx

Output:
PASS
or
SUSPECT: <aspect-label-1>, <aspect-label-2>, ...
```

The scanner forms its own first impression. If the fix didn't hold, it surfaces naturally; if a different bug exists, it is just as likely to be flagged.

**Why invoke the skills inside the SubAgent rather than paraphrase them in the prompt?** Skills carry the full methodology — paraphrasing into bullets loses fidelity and depends on the dispatcher having read the skill at all. Skills are stable, not contaminating: they describe *how to evaluate*, not *prior verdicts on this item*, so loading them does not violate blinding. The blinding rule is specifically about **history and outcomes**, not methodology.

## Stage 2 — aspect labels only

When Stage 1 returns `SUSPECT: error handling, naming`, Stage 2 receives the same skill-invocation instruction plus the aspect labels:

```
Invoke /coding:review and /coding:testing via the Skill tool to load
the review criteria.

Then focus your inspection on these aspects: error handling, naming.
Re-derive findings from scratch — do not rely on any prior reviewer's
notes.
```

Not: `Stage 1 found unhandled null on query prop at line 42`. Stage 2 must re-derive the specific finding, otherwise it just rubber-stamps Stage 1.

## Fixer — NOT blind to findings, still loads the skill

Once Stage 2 returns FAIL with a specific finding, the Fixer prompt **does** receive the full context — file path, line numbers, quoted issue, and any prior-cycle constraints needed to avoid regression (e.g., "preserve existing prop APIs — do not rename"). Fixers act on findings; blinding them would force them to re-inspect and slow the loop. The Fixer prompt still instructs it to invoke review_skills via the Skill tool, so the fix is performed under the same methodology that flagged the issue.

## Dispatcher checklist

Before sending any Review/Sample SubAgent prompt, confirm it includes:

- An explicit instruction to invoke each review_skill via the Skill tool (e.g., "Invoke `/coding:review` via the Skill tool"). Just naming the skill is not enough.

Then scan it for these strings and remove them:

- `Review = done|failed|pending`, `Execute = done`, `sample_passes`
- "previously", "last cycle", "cycle N", "prior", "recheck"
- "fixer", "patched", "fixed by", "just fixed"
- "Stage 1 said", "Stage 2 said", "PASS", "FAIL", "SUSPECT" (when referring to past verdicts)
- specific line numbers or code excerpts from the Notes/Log columns

If any survives, the prompt is leaky. Strip it before dispatch.
