# claude-flows

A 3-command plugin that makes Claude Code's autonomous primitives easy to reach for and hard to misuse.

**Status:** v0.1.0 scaffold. No commands implemented yet. See [PLAN.md](./PLAN.md) for the full design and phased roadmap.

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

## Install (when v0.1.0 ships)

```bash
claude --plugin-dir .
# or symlink:
ln -s "$PWD" ~/.claude/plugins/claude-flows
```

Currently nothing to run — see [PLAN.md](./PLAN.md) for what's coming and in what order.

---

## License

MIT
