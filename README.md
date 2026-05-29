# claude-flows

A 3-command plugin that makes Claude Code's autonomous primitives easy to reach for and hard to misuse.

**Status:** v0.2.0 — `/flow-goal` shipped (Phase 1). `/flow-fork` and `/flow-workflow` deferred until `/flow-goal` has been in real use. See [PLAN.md](./PLAN.md) for the full design and phased roadmap.

---

## What this is

Claude Code ships several autonomous primitives — `/goal`, `/branch`, worktrees, and (as of 2026-05-28) dynamic workflows. They are powerful but each has a learning curve, and the failure modes show up only on the second or third use.

`claude-flows` wraps them with opinionated defaults, safety pre-flights, and the compound shapes people actually want:

- **`/flow-goal`** — turns a natural-language task description into a well-shaped `/goal` condition the evaluator can actually verify.
- **`/flow-fork`** — combines `/branch` + `git worktree add` + a captured reason note, so future-you can find your way back to why this fork exists.
- **`/flow-workflow`** — adds a scope + cost pre-confirmation before dispatching a dynamic workflow.

This plugin does **not** try to be a Ralph loop, a planning system, or a dashboard. It is a thin layer over what Claude Code already does.

---

## Relationship to ralph

[`ralph`](https://github.com/flight505/ralph) and `claude-flows` are sister plugins addressing different problems:

| | ralph | claude-flows |
|---|---|---|
| **Who plans the work** | You (`plan.md`) | Claude (via `/goal` or workflows) |
| **Where the loop lives** | Shell script | Inside a Claude Code session |
| **Primary primitive** | `claude -p` in bash | `/goal`, `/branch`, dynamic workflows |
| **Best for** | Plan-driven, transparent, sub-Max-tier runs | Goal-driven, exploratory, codebase-scale work |
| **Tier** | Any | Pro+ for `/goal`/`/branch`, Max+ for workflows |

If you have a checklist you want executed shell-style, use ralph. If you want to lean on Claude Code's native autonomous primitives, use this.

---

## Install

```bash
claude --plugin-dir .
# or symlink:
ln -s "$PWD" ~/.claude/plugins/claude-flows
```

## Usage — `/flow-goal`

The only command shipped today. Turns a natural-language task into a well-shaped `/goal` condition the evaluator can actually verify, shows you the draft, and on `y` invokes `/goal` with it.

```
/flow-goal migrate all axios calls to fetch and keep tests passing
```

The command:

1. Looks at the cwd to identify test runner, linter, and the scope of the task.
2. Asks at most one clarifying question if anything material is ambiguous.
3. Drafts a 2–4-clause condition with verification commands inline and a turn cap.
4. Presents the draft for `y / edit / cancel`.
5. On `y`, invokes `/goal`. The goal then runs across as many turns as it needs until the condition holds.

See [templates/goal-shapes.md](./templates/goal-shapes.md) for the patterns the command draws on (migration, refactor, bug-fix, test-fix, hardening, plan-driven, doc completion) and the anti-patterns it avoids.

See [PLAN.md](./PLAN.md) for what's coming next (and what's deliberately not).

---

## License

MIT
