---
name: check
description: Read-only "is everything still passing?" check. Runs the test-runner and linter subagents in parallel and gives one combined pass/fail readout. Does not fix anything.
disable-model-invocation: true
argument-hint: [target]
---

# Check

A fast, read-only green/red check. Running `/check` spawns the `test-runner` and
`linter` subagents in parallel, waits for both, and reports ONE combined verdict.
It never edits code, runs no auto-fix, and never commits — it only reports.

`$ARGUMENTS` = `<target>` (optional)
- `<target>`: the component or path to check (e.g. "backend", "the recruitment
  page", "src/auth"). If omitted, check the whole project (or the current
  uncommitted changes if that's the obvious scope).

## Steps
1. Spawn `test-runner` and `linter` in parallel, each scoped to the target. Let
   each work in its own context and return only its summary.
2. Wait for both to finish.
3. Combine their results into a single readout. LEAD with the overall verdict:

## Check: <target> — PASS / FAIL

- **Tests**: <N passed, M failed, K broken>
- **Lint**: <N errors, M warnings across F files>

If everything is green, say so in one line and stop.

If anything failed, under the verdict list only the essentials so the user can
see what's red at a glance:
- the failing/broken test names (with `file:line`)
- the top lint issues (most common rule first)

Then end with exactly one pointer line:
> To fix these, run `/fix-cycle <target>`.

## Rules
- Read-only. Never edit code, never run an auto-fix flag, never commit.
- If either subagent reports it couldn't run, mark that half of the check as
  "could not run" (not as a pass) and show the command/error it reported.
- Overall verdict is PASS only if BOTH tests and lint are clean and both actually
  ran. Any failure, broken test, lint error, or "could not run" makes it FAIL.
- Keep the whole readout short and scannable — this is a spot check, not a report.
