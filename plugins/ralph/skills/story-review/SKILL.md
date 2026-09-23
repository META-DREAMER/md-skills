---
name: story-review
description: "Fresh-context adversarial review of Ralph-loop work before PR review. Runs at review checkpoints (stories flagged review:true, protected-surface diffs, or pre-PR) over the cumulative diff since the last review. Reports only high-confidence findings; a clean review is a successful review."
---

# Story Review

You are a fresh reviewer with no attachment to the code. Find what would otherwise surface in PR review, where each finding costs a triage, fix and reply cycle. Assume the diff is wrong until it proves otherwise, but **report only what you can defend**: a hallucinated finding costs an iteration to refute, or a "fix" that breaks green code. `REVIEW_CLEAN` is a normal outcome.

## Repo facts

**Repo facts.** Resolve each fact below in order: its `.agents/config.yaml` key; what the project's `CLAUDE.md`, `AGENTS.md` or README names; the conventional places listed. Relative paths resolve from the repo root. A fact you only read and cannot find is skipped with a note; one you must write is created at the default shown. Never guess a command. Say what you resolved, and from where, in your first status line.

| Fact | Key | Look for | If none |
| --- | --- | --- | --- |
| Sprint folders | `sprint_root` | an existing `sprints/` | create `sprints/` |
| Rule docs | `rule_docs` | root and package `CLAUDE.md` / `AGENTS.md`, and the docs they route to | root `CLAUDE.md` or `AGENTS.md` |
| Loop cursor | `ralph.state_file` | an existing `ralph/.state` | create `ralph/.state` |

## Inputs

- **Sprint** (e.g. `076-position-lifecycle`) plus one scope:
  - **Checkpoint** (the loop default): `git diff <lastReviewedSha>`, with `lastReviewedSha` from `ralph.state_file` (absent → the default branch). Map stories from `git log <lastReviewedSha>..HEAD --oneline`. **First check the anchor is an ancestor of HEAD** (`git merge-base --is-ancestor <sha> HEAD`): a rebase orphans the stored SHA. Orphaned → use the most recent checkpoint commit in `git log <default-branch>..HEAD`.
  - **Single story** (e.g. `076-12`): its commits plus uncommitted changes.
  - **Whole branch**: `git diff <default-branch>`, the pre-PR sweep.
- Recompute the file list from git; don't trust a passed list.
- Materialize the scope once: one `git diff <anchor>` (to a temp file if large) and one `git log --oneline`. Per-file `git show` calls rebuild the same diff at several times the cost.

## Judge against

1. Each covered story's `acceptanceCriteria` and `description` (`spec-json` skill).
2. The `rule_docs` and the owning package's own rules: operating rules, named failure modes, quality bars.
3. The sprint's `lessons.md`. A diff repeating a recorded lesson is an automatic finding.

## Pending stories

Before calling anything missing, load the stories with `passes != true`. **Don't report a gap a pending story explicitly owns.** Exception: an intermediate state that is dangerous now (an invariant broken on a live path touching money, idempotency, auth or credentials); report it and name the story you checked. Suppressed findings are not reported at all.

## Passes

One pass at a time over the full diff. Collect everything, including uncertain candidates; verification filters.

1. **Correctness.** For each changed function, construct the input or state that breaks it: boundaries, null or absent values, error paths leaving state inconsistent, interleaving at each `await` in a single-threaded actor, replays and duplicate deliveries.
2. **Invariants** the `rule_docs` declare for the area. Common shapes even when unstated: exact-precision values stay exact end-to-end; idempotency fingerprints cover every user-controlled input; decode branches fail closed; claims are atomic before external I/O; scheduled work is never left past-due; credentials never reach logs or responses; identifiers are normalized before keying or encoding; wire values are parsed into domain types at the boundary.
3. **Tests.** Name the test that fails if each behavioural change is reverted; none is a finding. Tests assert durable behaviour (rows, payloads, transitions), not mock-call counts. Fixtures didn't weaken production types.
4. **Blast radius.** Dependents of changed shapes, the scope a repo-wide typecheck covers: other consumers of renamed or retyped exports, fixtures derived from a changed schema, enum mirrors, config and test-config parity for new bindings or env. In checkpoint mode this matters most across stories: story A's schema change against story C's consumer.

Style, dead code, logging, comments and naming are `ralph-polish`'s job. Skip them.

## Verify before reporting

Try to refute each candidate: re-read callers, look for an upstream guard, check pending stories, run the test if cheap. Then classify:

- **CONFIRMED**: you can state the failing input or state and trace it to the wrong outcome. Only these are fix requests.
- **PLAUSIBLE**: not refuted, not confirmed, and touching an invariant (exact values, idempotency, auth, data loss, credentials). These go to `handoff.concerns` or `FOLLOWUP.md`. A PLAUSIBLE off-invariant is dropped.

Never report: style no rule backs, hypotheticals without a concrete trigger, what the quality gate enforces, "consider adding" suggestions, gaps a pending story owns, or lessons the diff doesn't violate.

## Output

Raw findings, most severe first, no preamble:

```
REVIEW_FINDINGS
Scope: <lastReviewedSha>..HEAD — stories 076-10, 076-11, 076-12
1. [CONFIRMED][correctness] packages/<pkg>/src/<file>.ts:42 — <one-sentence defect>.
   Failure: <concrete inputs/state → wrong outcome>. Fix: <suggested change>.
2. [PLAUSIBLE][invariants] ... (for handoff.concerns, not a fix)
```

or:

```
REVIEW_CLEAN
Scope: <lastReviewedSha>..HEAD — stories 076-10, 076-11, 076-12
```

Don't edit files or commit. The builder decides what to fix.
