# claude-flows

Helper plugin for Claude Code's autonomous primitives — `/goal`, `/branch`, worktrees, dynamic workflows.

---

## Why this exists

On 2026-05-28 Anthropic shipped dynamic workflows in Claude Code. Combined with the existing `/goal`, `/branch`, and worktree primitives, Claude Code now has a strong autonomous-execution layer built in. But the primitives have learning curves and footguns:

- `/goal` only works if the condition is verifiable from conversation output, multi-part, and bounded. Most users write a one-line condition that either never resolves or resolves prematurely.
- `/branch` captures the conversation fork but loses the *reason* for the fork five days later.
- Dynamic workflows "consume substantially more tokens than a typical Claude Code session" — and the native trigger is just "ask Claude to create a workflow," with no pre-flight.

`claude-flows` is the thin wrapper layer that fixes these papercuts. It does not invent new primitives, it does not replace the native ones, and it does not try to be a Ralph loop.

## Relationship to ralph

[`../ralph/`](../ralph/) is the sister plugin. They address different problems and should not converge.

- **ralph** — user-authored plan, executed by a bash loop, shell-supervised. The user knows what they want done and wants it executed.
- **claude-flows** — user describes a task, plugin shapes it into the right Claude Code primitive (`/goal`, `/branch`, workflow), runs in-session. The user wants Claude to plan or judge done-ness.

If a feature ever feels like it could go in either plugin, it almost certainly belongs in neither — it's drift.

---

## Design discipline (same as ralph)

The same 3-command budget. The same anti-patterns. The same "if it doesn't earn its place, it doesn't ship."

- **At most 3 slash commands.** Non-negotiable.
- **No hooks, no agents, no MCP servers, no skills** unless one of them is the strictly simpler answer to a problem a command would solve.
- **No wizards, no multi-step interactive setup.** A command answers one question, takes 1–2 clarifying inputs at most, and acts.
- **No state files beyond what the native primitives already require.** `.flows/forks/<name>.md` is borderline acceptable for `/flow-fork` because it captures information the native `/branch` discards.
- **No DSL for goals, scopes, or workflows.** If a config grows past a handful of env vars, the design is wrong.

## Stack

- Bash 3.2-compatible (macOS default) where shell is unavoidable.
- Slash-command markdown files that drive Claude to do the work where the work is mostly conversation-shaped.
- `jq` for JSON. Nothing else without justification.

## Read before extending

Before adding a command, hook, or skill:

1. Check whether the native Claude Code primitive (`/goal`, `/branch`, worktrees, `ultracode`, dynamic workflows) already solves it. If yes, don't wrap it just to "make it discoverable."
2. Check whether the addition crosses into ralph's territory (plan files, shell loops, fresh-context iteration). If yes, it belongs in ralph or in neither.
3. Read the [Claude Code dynamic workflows blog post](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) and decide whether the addition is better done as a workflow Claude plans itself than as a fixed command.

See [PLAN.md](./PLAN.md) for the implementation roadmap and the deferred list.

---

**Status:** v0.2.0 — Phase 1 (`/flow-goal`) shipped. Phases 2 and 3 deferred until `/flow-goal` has been in real use.
**Maintained by:** Jesper Vang (@flight505)
