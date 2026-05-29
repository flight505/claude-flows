# /goal shapes

Reference patterns for well-shaped `/goal` conditions. Loaded into context by `/flow-goal` when drafting.

## Rules every good goal follows

1. **Multi-clause.** 2–4 clauses. Single-clause goals fire too early because the evaluator only has one signal to read.
2. **Verifiable from output.** Each clause is a concrete check the evaluator can confirm from the conversation: a command exit status, a `git grep` count, a `git status` line, a file existence. Not vague predicates like "tests pass" or "code is clean."
3. **Bounded.** Always include `or stop after N turns`. Typical: 15 for surgical fixes, 25–30 for refactors, 50 for plan-driven work.
4. **Constrained.** At least one clause should pin down what must NOT change. `git status --porcelain` being empty is the safe default. For scoped work, name the directory: "no files outside `src/auth/` are modified."

## Patterns

### Migration (replace X with Y)

```
/goal all of the following hold:
  1. `git grep -l "from .axios" src/` returns nothing
  2. `npm test` exits 0
  3. `git status --porcelain` is empty
or stop after 30 turns
```

### Refactor (extract or restructure existing code)

```
/goal all of the following hold:
  1. every function previously in src/legacy.ts has been moved to a module under src/services/
  2. src/legacy.ts has been deleted
  3. `npm test` exits 0
  4. `tsc --noEmit` exits 0
or stop after 25 turns
```

### Bug-fix (with reproducer)

```
/goal all of the following hold:
  1. `npm test -- --testNamePattern 'login redirect'` exits 0 (this currently fails)
  2. `npm test` exits 0 for the full suite
  3. no files outside src/auth/ are modified
or stop after 15 turns
```

### Test-fix (make failing tests pass)

```
/goal all of the following hold:
  1. `pytest tests/test_auth.py` exits 0
  2. only files in src/auth/ and tests/ have been modified
  3. `ruff check src/auth/` exits 0
or stop after 20 turns
```

### Hardening pass (validation, error handling, security checks)

```
/goal all of the following hold:
  1. every public function in src/api/ has explicit input validation that raises on invalid input
  2. at least one new test case per function exercises an invalid-input path
  3. `pytest` exits 0
  4. `git diff src/api/ | wc -l` is under 500
or stop after 25 turns
```

### Plan-driven (work through a checklist)

```
/goal all of the following hold:
  1. every `- [ ]` item in PROJECT_PLAN.md has been changed to `- [x]`
  2. `git log --oneline main..HEAD | wc -l` is at least N (where N = original item count)
  3. `npm test` exits 0
or stop after 50 turns
```

### Doc completion

```
/goal all of the following hold:
  1. every exported function in src/api/ has a JSDoc block with @param and @returns
  2. `npm run lint:docs` exits 0
  3. `git status --porcelain` shows changes ONLY in src/api/
or stop after 30 turns
```

## Anti-patterns — do not copy these

| Bad clause | Why it's bad | Fixed |
|---|---|---|
| `the migration is complete` | Unverifiable | `git grep -l "from .axios" src/` returns nothing |
| `tests pass` | No command specified | `npm test` exits 0 |
| `all TODOs are resolved` | "Resolved" vs "deleted" is ambiguous | `git grep -c "TODO" src/` returns 0 AND `npm test` exits 0 |
| `the bug is fixed` | No reproducer | `npm test -- --testNamePattern '<failing test name>'` exits 0 |
| `the code is clean` | Subjective | `eslint src/` exits 0 AND `prettier --check src/` exits 0 |
| `at least one test passes` | Too permissive | `npm test` exits 0 (entire suite) |
| (omitting turn cap) | Goal runs until manually killed | append `or stop after N turns` |
| (single clause) | Fires too early | add `git status --porcelain is empty` as safety clause |

## When to ask the user a clarifying question

Ask ONLY if the answer would change which clauses you draft. Bias strongly toward NOT asking.

| Situation | Ask? | What to ask |
|---|---|---|
| Multiple test runners present (Jest + Mocha both configured) | Yes | "Which test runner should the goal check — Jest or Mocha?" |
| Task is "clean up X" with no defined success criterion | Yes | "What does 'cleaned up' mean here — fewer than N lines, tests still passing, something else?" |
| Task is "migrate Y to Z" with no scope hint and the codebase is large | Yes | "Migrate everywhere, or only in a specific directory?" |
| User said "fix the failing test" and there's exactly one failing test | No | Use the test name directly |
| User specified everything you need to draft 3 clauses | No | Draft directly |
| Test runner is obvious from a single config file | No | Default to it |

**Hard cap:** one clarifying question is acceptable, two is the maximum, three is a wizard.
