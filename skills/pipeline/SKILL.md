---
name: pipeline
description: Full gated pipeline. Fans out the research, review, and verify subagents on a target, consolidates findings, plans fixes for your approval, then implements the approved plan and stops before commit. You gate the plan and the diff; you commit.
disable-model-invocation: true
argument-hint: [target] [all|security|perf|quick]
---

# Pipeline

A comprehensive, gated loop:
**Gather (research + review + verify) -> Consolidate -> Plan -> [your gate] -> Fix -> [your gate] -> you commit.**

This is the big sibling of `/fix-cycle`. Where `/fix-cycle` runs only the light
checks (perf, lint, tests) before fixing, `/pipeline` also runs the researchers
and the perf/HIPAA reviewers for a full picture first. It edits nothing until you
approve the plan, and it never commits.

`$ARGUMENTS` = `<target> [focus]`
- `<target>`: component or path (e.g. "backend", "the recruitment page",
  "src/auth"). If omitted, ask which target before doing anything.
- `[focus]` (optional): narrow the pass for speed. One of:
  - `all` (default) — research + review + verify
  - `security` — auth-researcher + hipaa-reviewer + verify
  - `perf` — perf-reviewer + verify
  - `quick` — review + verify only (skip the researchers)

## Stage 1 — Gather (read-only, all in parallel)
Spawn the read-only subagents for the chosen focus, each scoped to the target,
and let them run in parallel. Wait for all of them.
- **Research** (skip for `quick`/`perf`): `auth-researcher`,
  `database-researcher`, `api-researcher`
- **Review**: `perf-reviewer` and/or `hipaa-reviewer` (per focus; `all` runs both)
- **Verify** (always): `test-runner`, `linter`

Each returns only its own summary — that's the point of fanning out, so the noisy
output stays in their contexts.

## Stage 2 — Consolidate
Merge everything into ONE picture, organized as:
- **State**: the test-runner + linter verdict (pass/fail, failing/broken tests,
  top lint issues)
- **Issues to fix**: deduplicated findings from perf-reviewer + hipaa-reviewer,
  each with `file:line` and severity, most serious first
- **Context**: only the structural notes from the researchers that actually bear
  on the fixes (keep it short)

If verify is green AND no issues were flagged, report the clean result and STOP —
there's nothing to fix.

## Stage 3 — Plan (optimizer, plan only)
Hand the consolidated Issues (plus the relevant Context) to the `optimizer`
subagent in PLAN mode ("plan only, do not edit any file"). Present its fix plan
exactly as returned.

## Stage 4 — STOP for review (Gate 1)
Do NOT edit any file. Do NOT start the optimizer's code phase. End the turn with
the plan followed by:

> Plan ready for your review. Reply "approved" to implement, or tell me what to change.

Then wait for the user.

## After the user approves the plan
Only once the user approves:
1. Invoke the `optimizer` in CODE mode to implement EXACTLY the approved plan.
2. Re-run `test-runner` and `linter` to confirm the fixes hold and nothing
   regressed.
3. Present the diff and the fresh verify results (Gate 2). Do NOT commit or push.
4. End with:
   > Changes applied and verified — review the diff above and commit when you're ready.

The user commits themselves.

## Rules
- **Never commit or push.** Commit is always the user's manual step.
- **Gate 1 is mandatory** — no file is edited before the user approves the plan.
- Only the `optimizer` may write, and only after approval. Every other stage is
  read-only.
- If the approval comes with changes, implement the amended plan, not the original.
- If any agent reports it couldn't run, surface that plainly and don't treat a
  failed check as a pass.
- This pass fans out many agents and can use substantially more tokens than
  `/fix-cycle`. For a quick green/red use `/check`; for a light fix loop use
  `/fix-cycle`; reach for `/pipeline` when you want the full picture.
- Keep the consolidated report and the plan tight and scannable.
