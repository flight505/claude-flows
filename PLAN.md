# claude-flows — implementation plan

**One-line:** A 3-command plugin that makes Claude Code's autonomous primitives (`/goal`, `/branch`, worktrees, dynamic workflows) easy to reach for and hard to misuse.

**Design constraints (same as ralph):**
- At most 3 slash commands. Non-negotiable.
- No hooks, no agents, no MCP servers, no skills.
- No wizards, no multi-checkpoint setup.
- No state files beyond what the native primitives already require.
- No DSL.

---

## The 3 commands

### `/flow-goal <task description>`

**Problem it solves.** The hardest part of `/goal` is writing the condition. The fast-model evaluator judges the condition against what Claude has surfaced in the conversation. That means the condition has to be:

- **Verifiable from output** — "the migration is done" is unverifiable; "`git grep -l 'from .axios'` returns nothing" is verifiable.
- **Multi-part** — one clause rarely covers the real success criteria.
- **Bounded** — include a turn cap (`or stop after 30 turns`) so it doesn't run forever.

Most users get this wrong on the first try. `/flow-goal` removes that failure mode.

**Behavior.**

```
> /flow-goal migrate all axios calls to fetch and keep tests passing

flows: Detected — Jest in package.json, ESLint in .eslintrc, src/ contains 14 axios imports.
flows: One question — also require zero new console.error in test output, or just `npm test` exit-0?
> just exit-0

flows: Proposed goal:

  /goal all of the following hold:
    1. `git grep -l "from .axios" src/` returns nothing
    2. `npm test` exits 0
    3. `git status --porcelain` is empty
  or stop after 30 turns

  Continue? [y / edit / cancel]
> y

flows: invoking /goal …
```

**Implementation.**

A slash command markdown file. The body instructs Claude (in the active session) to:

1. Read the user's task description from `$ARGUMENTS`.
2. Inspect the cwd to identify verification surfaces — test runner, linter, build tool, code patterns relevant to the task.
3. Ask at most 1–2 clarifying questions if anything material is ambiguous. Otherwise skip.
4. Draft a 2–4-clause condition with verification commands inline and a turn cap.
5. Present the draft to the user with `[y / edit / cancel]`.
6. On `y`, invoke `/goal` with the drafted condition.
7. On `edit`, accept user edits, re-present.
8. On `cancel`, exit cleanly.

No bash script needed. The command body is conversation-shaped.

**Templates** (in `templates/`):

- `templates/goal-shapes.md` — concrete examples of well-shaped multi-clause goals for common patterns: migration, refactor, bug-fix, test-fix, hardening pass. Loaded into context by the slash command.

**Verification gates before shipping `/flow-goal`:**

- Invoking with a clear migration task produces a 3-clause goal with at least one verification command.
- Invoking with an ambiguous task ("clean up the auth code") triggers exactly one clarifying question, not three.
- The drafted goal always includes a turn cap.
- `cancel` exits without invoking `/goal`.

---

### `/flow-fork <one-line name>`

**Problem it solves.** Native `/branch` forks the conversation. Native `git worktree add` forks the filesystem. The compound — *I want both, named the same, with a note about why* — is what people actually do when they want to explore an alternative without losing the current path. The reason note is the part native `/branch` discards, and which Future You always wishes existed.

**Behavior.**

```
> /flow-fork try-rest-api-rewrite

flows: Why this fork? (one line, captured for future-you)
> the GraphQL approach feels fragile, want to see if REST is cleaner

flows: Created:
  ├── conversation branch: try-rest-api-rewrite
  ├── git worktree:        .claude/worktrees/try-rest-api-rewrite
  └── reason file:         .flows/forks/try-rest-api-rewrite.md

flows: You are now in the branch. Use /resume to return to the parent session.
```

**Optional flag:** `--no-worktree` to fork the conversation only, skipping the filesystem branch.

**Implementation.**

A slash command that:

1. Validates `$ARGUMENTS` is a single kebab-case name.
2. Asks for the one-line reason (skip if it was passed via `--reason "…"`).
3. Invokes `/branch <name>`.
4. Runs `git worktree add .claude/worktrees/<name>` unless `--no-worktree` was set.
5. Writes `.flows/forks/<name>.md` with: timestamp, parent session ID, reason, optional commit SHA at fork time.
6. Confirms all three operations and reminds the user how to navigate back.

`.flows/` should be added to `.gitignore` automatically on first use.

**Verification gates:**

- Invoking creates all three artifacts.
- `--no-worktree` skips only the worktree step.
- Repeated invocation with the same name refuses and points at the existing fork.
- Reason file contains parseable frontmatter (so /flow-status or a future command can read it).

---

### `/flow-workflow <task description>`

**Problem it solves.** Dynamic workflows "consume substantially more tokens than a typical Claude Code session" ([Anthropic blog](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code)). The native trigger is "ask Claude to create a workflow" or enabling `ultracode` — no pre-flight estimate of scope, cost, or what gets touched. The first time you trigger a workflow on a real codebase, the absence of a pre-flight matters.

`/flow-workflow` adds the pre-confirmation.

**Behavior.**

```
> /flow-workflow find all SQL injection vulnerabilities and propose fixes

flows: This will use dynamic workflows. Pre-flight:

  Task scope:       .  (current repo)
  File patterns:    *.py, *.ts, *.js, *.go  (auto-detected)
  Will NOT touch:   node_modules, .venv, dist, build, *.lock, .git
  Est. subagents:   20–50 (Claude decides at plan time)
  Est. token cost:  ~10–50× a normal session (could be $5–$50 on Max)
  Effort level:     ultracode (xhigh)
  Kill switch:      Esc, or close terminal

  Proceed? [y / edit-scope / cancel]
> y

flows: enabling ultracode and dispatching workflow…
```

**Implementation.**

A slash command that:

1. Inspects the cwd to identify file patterns and detect a project type.
2. Constructs a scope pre-flight using sensible defaults from `templates/workflow-scope.md`.
3. Presents the pre-flight with `[y / edit-scope / cancel]`.
4. On `edit-scope`, lets the user adjust patterns/exclusions and re-presents.
5. On `y`, sets the session effort to `ultracode` and submits `create a workflow for: <task>` with the scope hints prepended.
6. On `cancel`, exits cleanly without enabling `ultracode`.

The estimates are conservative defaults — *not* magic. They are better than zero pre-flight info. We document explicitly that they're best-guess.

**Verification gates:**

- Invoking on a Python repo detects `.py` and ignores `.venv`.
- Invoking on a JS repo detects `*.ts/*.js` and ignores `node_modules`.
- `cancel` does not modify `ultracode` or session state.
- `edit-scope` round-trips correctly.

---

## Deliberately not in scope

| Idea | Why not |
|---|---|
| **Plan files / checklists** | Ralph's job. Different problem. Putting this in claude-flows is drift. |
| **Cron / scheduling** | Use `/loop` (in-session) or routines (cloud) directly. Both already exist. |
| **`/rewind` wrapper** | Native `/rewind` is fine. There's no compound shape worth wrapping. |
| **Worktree dashboard or list** | `git worktree list` works. Adding UI is the harness trap. |
| **Cost / usage dashboards** | `/usage` exists. |
| **`/flow-status` showing active goals + branches + workflows** | Tempting but drifts into dashboard. Revisit only if missing-state-overview is reported as friction. |
| **Auto-triggering skill that intercepts `/goal`** | Skills count against the surface budget, and a hidden interceptor is harder to reason about than an explicit command. The command form is more honest. |
| **Multi-attempt parallel runs (spawn N forks, run same task, pick winner)** | Compelling but operationally complex. Revisit only on real demand. |
| **Integration with `ralph`** | They address different problems. Crossing them dilutes both. |

---

## Phased rollout

### Phase 0 — scaffold (this commit)

- Repo structure created.
- `plugin.json`, `README.md`, `CLAUDE.md`, `PLAN.md`, `.gitignore` in place.
- No commands implemented.
- `commands/`, `lib/`, `templates/` directories exist but are empty.
- **Marketplace entry**: `claude-flows` registered at v0.1.0 in the parent marketplace.

### Phase 1 — `/flow-goal` only ✓ shipped in v0.2.0

- ✓ `commands/flow-goal.md` — conversation-shaped slash command (no bash). Procedure: inspect cwd, ask ≤1 clarifying question, draft 2–4 clauses with turn cap, present `y/edit/cancel`, invoke `/goal` on confirm.
- ✓ `templates/goal-shapes.md` — 7 patterns (migration, refactor, bug-fix, test-fix, hardening, plan-driven, doc completion) plus an anti-patterns table and a "when to ask a clarifying question" decision table.
- ✓ `plugin.json` `commands` array populated; claude-flows bumped to v0.2.0.
- ✓ `marketplace.json` claude-flows entry version bumped.
- **Manual verification gates** (not auto-runnable — to be confirmed by the user on a real task before declaring Phase 1 fully proven):
  - Invoking on a clear migration task produces a 3-clause goal with at least one verification command.
  - Invoking on an ambiguous task triggers ≤1 clarifying question, not three.
  - Drafted goal always includes a turn cap.
  - `cancel` exits without invoking `/goal`.

### Phase 2 — `/flow-fork` (deferred)

Build only after Phase 1 has been in real use long enough to be sure the goal-shape captured by `/flow-goal` is right. Do not build pre-emptively.

### Phase 3 — `/flow-workflow` (deferred)

Build only after Phase 1 (and ideally Phase 2). Requires real use of dynamic workflows to know what the pre-flight should actually contain. Trying to design it before you've used the underlying feature is the wizard trap.

---

## Open design questions for later

These are decisions to make when the relevant phase starts, not now:

- **Should `/flow-goal` accept multiple goal styles?** Some users will want "do this until tests pass," others "complete every TODO in plan.md." A single command is fine if the templates cover both. Revisit if Phase 1 use shows divergent shapes.
- **Where do fork reason files live?** `.flows/forks/<name>.md` is the current plan. Should this be `.claude/forks/`? Decide at Phase 2 start, based on whether the user wants `.flows/` to be a recognized "this is a claude-flows project" marker.
- **Should `/flow-workflow` estimates ever be real?** The current plan is "documented best-guess defaults." We could integrate with `/usage` to track actual spend per workflow and refine estimates over time. Probably premature for v0.1.

---

**Status:** Phase 1 shipped in v0.2.0 (`/flow-goal` + `goal-shapes.md`). Phases 2 and 3 deferred per the design — do not build them until `/flow-goal` has been in real use and produced concrete signal about what their final shape should be.
