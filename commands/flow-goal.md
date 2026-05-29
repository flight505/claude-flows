---
description: Turn a natural-language task into a well-shaped /goal condition the evaluator can actually verify.
---

The user invoked `/flow-goal` with this task description: `$ARGUMENTS`

If `$ARGUMENTS` is empty, tell the user the usage is `/flow-goal <task description>` (e.g. `/flow-goal migrate all axios calls to fetch and keep tests passing`) and stop without doing anything else.

## What this command does

Turns the user's natural-language task into a well-shaped `/goal` condition, then invokes `/goal` with it. The hardest part of `/goal` is the condition — most users write something unverifiable, single-clause, or unbounded, and the goal either never resolves or resolves prematurely. This command exists to remove that failure mode.

## Before drafting

Read the patterns file at `${CLAUDE_PLUGIN_ROOT}/templates/goal-shapes.md`. Use the patterns as references; do not copy them verbatim. The anti-patterns table is what to avoid.

## Procedure

### 1. Inspect the cwd briefly

Identify verification surfaces. Spend fewer than 5 tool calls on this — the goal is to know which commands to put in the condition, not to map the whole project.

Look for:
- **Test runner**: `package.json` (Jest/Vitest/Mocha), `pytest.ini` or `[tool.pytest]` in `pyproject.toml`, `go.mod`, `Cargo.toml`, a `Makefile` with a `test` target.
- **Linter / type checker**: `.eslintrc*`, `ruff.toml`, `tsconfig.json`, `golangci.yml`, `mypy.ini`.
- **Build tool**: `package.json` scripts, `Makefile`, `cargo build`, `go build`.
- **Code patterns relevant to the task** — e.g., for "migrate axios to fetch," run `git grep -l "from .axios" src/ | wc -l` to count affected files. This tells you whether the scope is 3 files or 300.

### 2. Ask at most ONE clarifying question

Two is the hard cap. Bias strongly toward NOT asking. Only ask when the answer would change which clauses you draft.

Examples of when to ask:
- Multiple test runners are configured and there's no obvious primary.
- The task is "clean up X" with no defined success criterion.
- The task could mean "everywhere" or "only in this subdirectory" and the codebase is large.

Examples of when NOT to ask:
- You can reasonably default (e.g., always include `git status --porcelain` as a safety clause).
- The user has been specific.
- One test runner is configured.

### 3. Draft a condition

Always 2–4 clauses. Always include a turn cap. Always include at least one constraint clause (the safe default is `git status --porcelain is empty`). Match the shapes in `goal-shapes.md`.

### 4. Present the draft in this exact format

```
Proposed goal:

  /goal all of the following hold:
    1. <clause 1 with a concrete command>
    2. <clause 2 with a concrete command>
    3. <clause 3 with a concrete command>
  or stop after <N> turns

Continue? [y / edit / cancel]
```

### 5. Act on the user's reply

- **`y`** (or yes, ok, sure, go, etc.): invoke `/goal` with exactly the drafted condition. After invoking, say nothing further — the goal is now active and Claude Code will manage the loop. The user can run `/goal` with no arguments at any time to see status.
- **`edit`** (or specific edits like "make clause 2 use `pnpm test` instead"): accept the user's revisions, re-present the updated draft in the same format, wait again.
- **`cancel`** (or no, stop, n, abort): exit cleanly. Do not invoke `/goal`. Do not enable anything. Do not leave state behind.

## Hard rules

- Never invoke `/goal` without showing the draft first. The pre-flight is the entire point.
- Never exceed 2 clarifying questions in total.
- Never draft a single-clause condition.
- Never omit the turn cap.
- Never propose vague clauses like "tests pass" or "the code is clean" — every clause needs a specific command or file pattern.
