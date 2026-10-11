# Claude Code Agent Pipeline

Eight custom subagents and three slash commands for [Claude Code](https://claude.com/claude-code) that review a codebase, plan fixes, and apply them only after I approve the plan. Nothing in here commits or pushes. That step stays with me.

I wrote these for my own projects because I wanted an AI workflow where most agents can't change anything, and the one that can has to stop and ask first.

## How it works

```
/pipeline <target>

  Gather (read-only, in parallel)
    research:  auth-researcher, database-researcher, api-researcher
    review:    perf-reviewer, hipaa-reviewer
    verify:    test-runner, linter
        |
  Consolidate into one deduplicated issue list
        |
  optimizer (PLAN phase)  ->  fix plan
        |
  GATE 1: I approve or edit the plan
        |
  optimizer (CODE phase)  ->  implements exactly the approved plan
        |
  test-runner + linter run again
        |
  GATE 2: I review the diff and commit myself
```

Each subagent runs in its own context and returns only a summary, so the long test logs and lint output never fill up the main conversation.

## The agents

| Agent | Tools | What it does |
| :--- | :--- | :--- |
| `api-researcher` | Read, Grep, Glob | Maps endpoints, request/response shapes, and which component calls which |
| `auth-researcher` | Read, Grep, Glob | Traces how a request proves identity and where access control is enforced |
| `database-researcher` | Read, Grep, Glob | Maps schema, models, migrations, and query patterns |
| `perf-reviewer` | Read, Grep, Glob | Returns at most five performance issues, ranked Urgent / Moderate / Can Wait |
| `hipaa-reviewer` | Read, Grep, Glob | Checks code against the HIPAA Security Rule technical safeguards and flags likely gaps |
| `test-runner` | Read, Grep, Glob, Bash | Runs the test suite and explains each failure in one sentence |
| `linter` | Read, Grep, Glob, Bash | Runs the project's linter without auto-fix and groups violations by rule and file |
| `optimizer` | Read, Grep, Glob, Edit, Write, Bash | The only agent that can edit files. Plans first, edits only after approval |

## The commands

| Command | What it runs |
| :--- | :--- |
| `/check [target]` | `test-runner` and `linter` in parallel, with one combined PASS / FAIL readout. Read-only |
| `/fix-cycle [target] [perf\|lint\|test\|hipaa\|all]` | The chosen reviewers, then the optimizer's plan, then stops for approval |
| `/pipeline [target] [all\|security\|perf\|quick]` | Everything: researchers, reviewers, verify, plan, approval gate, fix, re-verify |

All three commands set `disable-model-invocation: true`, so Claude never starts them on its own. I have to type them.

## Design decisions

**Least privilege per agent.** Five of the eight agents get only `Read`, `Grep`, and `Glob`. They cannot edit a file or run a command even if a prompt told them to. The test runner and linter get `Bash` because they have to execute something, but their instructions forbid auto-fix flags and edits.

**One writer, two phases.** Only the optimizer has `Edit` and `Write`. It defaults to the PLAN phase, where it outputs a fix plan and stops. It enters the CODE phase only after I approve, and then it implements exactly that plan with no extra refactors. If it hits something the plan didn't cover, it stops and reports instead of improvising.

**The human commits.** No agent or command runs `git commit` or `git push`. The pipeline ends by showing me the diff and the fresh test and lint results.

**"Could not run" is not a pass.** If a test suite or linter fails to start, the agent reports the command and the error verbatim. `/check` only returns PASS when both halves actually ran and came back clean.

**Fixed report formats.** Every agent has a required output structure with `file:line` references, so the consolidation step can merge and deduplicate findings from several agents.

**Agents say when they don't know.** The researchers put anything uncertain under "Open questions". The perf reviewer is told not to pad its list. The HIPAA reviewer has a "Can't tell from code" status and opens every report with a disclaimer that it is a technical review, not legal advice or a compliance certification.

## Install

Copy the agents and skills into your Claude Code config:

```bash
git clone https://github.com/FahedTaher/claude-code-agent-pipeline.git
cd claude-code-agent-pipeline
cp agents/*.md ~/.claude/agents/
cp -r skills/* ~/.claude/skills/
```

Then, in a project:

```
/check backend
/fix-cycle src/auth perf
/pipeline backend security
```

A few of the agent prompts name the kinds of components I was working with (FastAPI backend, browser extension, recruitment page) as examples. Edit those lines to match your own project.

## Repo layout

```
agents/           one Markdown file per subagent (frontmatter: name, description, tools, model)
skills/
  check/          SKILL.md for /check
  fix-cycle/      SKILL.md for /fix-cycle
  pipeline/       SKILL.md for /pipeline
```
