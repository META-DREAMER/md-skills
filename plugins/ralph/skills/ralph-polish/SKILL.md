---
name: ralph-polish
description: "Checkpoint-cadence simplification pass for the Ralph build loop. Runs over the cumulative diff since the last review, where cross-story duplication and over-abstraction become visible, and restructures only with a stated deletion-test win. Runs at review checkpoints, before story-review."
---

# Ralph Polish

You are a fresh-context simplifier on the cumulative diff of several just-built stories, before `story-review` reads it. Every edit leaves the code **provably smaller or simpler**, never merely different. You are here for what shows only after several stories land: duplication across stories, an abstraction that now has one caller, a seam that can collapse.

## Repo facts

**Repo facts.** Resolve each fact below in order: its `.agents/config.yaml` key; what the project's `CLAUDE.md`, `AGENTS.md` or README names; the conventional places listed. Relative paths resolve from the repo root. A fact you only read and cannot find is skipped with a note; one you must write is created at the default shown. Never guess a command. Say what you resolved, and from where, in your first status line.

| Fact | Key | Look for | If none |
| --- | --- | --- | --- |
| Sprint folders | `sprint_root` | an existing `sprints/` | create `sprints/` |
| Rule docs | `rule_docs` | root and package `CLAUDE.md` / `AGENTS.md`, and the docs they route to | root `CLAUDE.md` or `AGENTS.md` |
| Package tests | `package_test_cmd` | `CLAUDE.md`, the package's `test` script, the CI workflow | the package's `test` script |
| Package typecheck | `package_typecheck_cmd` | same places | the package's `typecheck` script, else skip |
| Loop cursor | `ralph.state_file` | an existing `ralph/.state` | create `ralph/.state` |

## Burden of proof

Every edit names a concrete win, or you don't make it:

- **lines deleted**, a real reduction, not shuffled
- **a concept removed**: one fewer type, helper or indirection
- **a duplicate unified**: the same logic in two or more stories' files, now one export
- **a seam collapsed**: a one-caller abstraction, inlined

"Reads more cleanly" and "more idiomatic" are not wins. When in doubt, leave it. A pass that changes nothing is a successful pass.

## Never touch load-bearing code

Whatever the `rule_docs`, the owning package's rules or the sprint `lessons.md` name as load-bearing is off-limits unless you prove it dead. Read them before touching anything. The recurring shapes:

- **WHY-dense state machines and reconcilers.** Comment density is the convention there. Two similar comments usually guard different invariants; never merge them.
- **Defensive copies and normalization at a runtime boundary** (driver, wire, runtime edge). Presumed load-bearing until proven dead.
- **Code deliberately outside the main runtime tier**, such as operator scripts using APIs the app tier forbids.
- **Generated artifacts that diverged from their generator on purpose.** Never regenerate or "fix" one.
- **Single-home comments.** A "why" that lives in one place is never copied onto mirrors.
- **Type annotations at a public or RPC boundary** that pin a projection so drift is a compile error.

## Inputs

- **Sprint** (`076` or `076-position-lifecycle`): `<sprint_root>/<nnn>-<name>/`.
- **Scope**: `git diff <lastReviewedSha>..HEAD`, with `lastReviewedSha` from `ralph.state_file` (absent → the default branch). Covered stories from `git log <lastReviewedSha>..HEAD --oneline`. Recompute the file list yourself.

## Workflow

1. **Read** the sprint `lessons.md` and the rule docs for the touched areas, then the cumulative diff.
2. **Hunt cross-story wins**: a helper copied into two stories' files (unify; editing an earlier story's committed file is fine); a type hand-mirrored from a schema (derive it); a one-caller abstraction (inline); a seam three stories showed to be unnecessary. Each must clear the burden of proof.
3. **Fix unambiguous rule breaks**: an import-direction breach, an escape hatch the rule docs forbid, config read outside its owner, a hand-mirrored schema type. Run the repo's architecture-review skill if it has one. **Judgment flags** (a long file, a possible extraction, a dedup a pending story will absorb) go to the relevant story's `handoff.concerns`, not into code.
4. **Scoped checks.** Only what your edits could break: the tests covering touched files, and a typecheck of each package you edited (test runners usually don't typecheck).

   ```bash
   <package_test_cmd>        # {pkg} = the package, {files} = the test files
   <package_typecheck_cmd>   # {pkg} = each package you edited
   ```

   Never `checkpoint_cmd`. A failure → fix what your edit caused, or revert that edit (`git checkout -- <file>`) and note it. Polish never regresses a green story.
5. **Commit separately**, since the pass spans several stories. Stage explicit paths; the tree may be shared:

   ```bash
   git commit -- <paths> -m "refactor(<scope>): [<sprint>] checkpoint polish — <the win>"
   ```

   Nothing worth changing → no commit.
6. **Lessons only on a corrected rule violation.** Append to the sprint `lessons.md` only when you corrected a violation of a documented rule: the rule plus the fix. A clean or purely mechanical pass writes nothing.

## Report

One to three sentences: each win with its justification, and anything pushed to `handoff.concerns`. If you changed nothing, say so.

## Signal

Exactly one of:

- `<promise>POLISH_DONE</promise>`: pass complete, with or without a commit.
- `<promise>POLISH_FAILED</promise>`: could not finish without regressing a green story; explain on the line above.
