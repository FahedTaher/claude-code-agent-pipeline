---
name: fix-cycle
description: Review a target, feed the findings to the optimizer's plan phase, and stop for your review before any code changes. A gated Plan -> Code -> Commit loop you trigger yourself.
disable-model-invocation: true
argument-hint: [target] [perf|lint|test|hipaa|all]
---

# Fix Cycle

A gated **Plan -> Code -> Commit** loop. Running `/fix-cycle` does Step 1 (review)
and Step 2 (plan) automatically, then STOPS for the user's review. It never edits
code or commits on its own — the user gates the plan, gates the diff, and commits.

`$ARGUMENTS` = `<target> [focus]`
- `<target>`: the component or path to work on (e.g. "backend", "the recruitment
  page", "src/auth"). If omitted, ask which target before doing anything.
- `[focus]` (optional): which review to run — `perf`, `lint`, `test`, `hipaa`, or
  `all`. Default when omitted: `all`.

## Step 1 — Review (gather issues)
Parse the target and focus from `$ARGUMENTS`. Spawn the matching read-only
reviewer subagent(s), in parallel when more than one, each scoped to the target:
- `perf`  -> `perf-reviewer`
- `lint`  -> `linter`
- `test`  -> `test-runner`
- `hipaa` -> `hipaa-reviewer`
- `all` or no focus -> `perf-reviewer`, `linter`, and `test-runner` together

Consolidate their reports into ONE deduplicated list of concrete issues, each
with `file:line` and a severity. If nothing was flagged, say so and stop — there
is nothing to fix.

## Step 2 — Plan (optimizer, plan only)
Hand the consolidated issue list to the **optimizer** subagent in PLAN mode
("plan only, do not edit any file"). Present its fix plan exactly as returned.

## Step 3 — STOP for review (Gate 1)
Do NOT edit any file. Do NOT invoke the optimizer's code phase yet. End the turn
with the plan followed by this line:

> Plan ready for your review. Reply "approved" to implement, or tell me what to change.

Then wait for the user.

## After the user approves the plan
Only once the user approves:
1. Invoke the **optimizer** in CODE mode to implement exactly the approved plan,
   then run the verification (tests/lint) named in the plan, and STOP before
   committing.
2. Present the diff and the verification results (Gate 2). Do NOT commit or push.
3. End with:
   > Changes applied and verified — review the diff above and commit when you're ready.

The user commits themselves.

## Rules
- **Never commit or push.** Commit is always the user's manual step.
- **Never skip Gate 1.** No file is edited before the user approves the plan.
- If the user's approval comes with changes, apply those to the plan before
  implementing — implement the approved version, not the original.
- If a reviewer or the optimizer reports it could not run, surface that plainly
  and stop rather than proceeding on partial results.
- Keep the consolidated issue list and the plan tight and scannable.
