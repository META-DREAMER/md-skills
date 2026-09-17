You are executing approved fixes for sprint `{{sprint}}` (a value of `none` means this PR is not tied to a sprint — see the sprint-vs-non-sprint note in step 3/4). You make code changes, run the quality gate, and commit. You may receive a single entry or a batch of related entries — apply all of them in one atomic commit.

## Context

Read the file(s) the entries point at and the `rationale` + `fixPlan` fields. Pull additional context only as needed:

- `{{rule_docs}}` — the project's coding rules and quality gate (from `.agents/config.yaml`; default: every `CLAUDE.md` in the repo)
- `{{state_dir}}/lessons.md` and the plan of record — sprint context and recent decisions (skip when sprint is `none`)
- `{{docs_root}}` — the project's spec docs (consult topically)

## Entries

```json
{{comment_entries_json}}
```

## What to do

Every step happens BEFORE the commit so HEAD lands clean.

1. **Implement all fixes at root cause** per each entry's `rationale` and `fixPlan`. Do not suppress, cast, or widen types. If a fix plan is wrong or stale given current code, update the approach — but keep each fix minimal and on-target. When an entry has a `secondOpinion` with `agree: false` and a `counterFixPlan`, prefer the `counterFixPlan` over the original `fixPlan`.

   **Don't gold-plate the fix.** The project's rule docs apply to your changes too: don't add comments to explain WHAT the code does, don't add defensive validation for scenarios that can't happen, don't add logs unless the fix specifically requires observability, don't introduce wrapper helpers for one-call-site abstractions. If the fixPlan calls for something that violates these rules (e.g. "add a TSDoc comment," "log the skipped entry"), implement the minimum that satisfies the actual concern — or if the entire fix would be noise, emit `<promise>BLOCKED</promise>` with rationale "fix violates anti-noise rules — should have been triaged invalid" and stop.

2. **Run scoped checks** on what the fixes touched — never repo-wide; the pre-commit hook lints and formats at commit, and PR CI typechecks dependents:

   Use the project's `package_test_cmd` from `.agents/config.yaml`, filled with the package and the test files the fix touches, plus that package's typecheck. When no config exists, use the command the project's own docs name for running one package's tests by file.

   If a check or the pre-commit hook fails, diagnose and fix. Try up to 2 attempts total (1 initial + 1 retry). If the second attempt still fails, emit `<promise>BLOCKED</promise>` with a short error excerpt and stop — do not commit. `<promise>DONE</promise>` means all three happened: gate green, review.json updated, commit made — the orchestrator reads the commit and the ledger, so a partial completion leaves it with an inconsistent state to unpick.

3. **Capture a durable lesson only if any fix surfaced a genuinely reusable pattern; skip if the fixes are one-offs.**
   - **Sprint branch** (`{{sprint}}` is a slug): append to `{{state_dir}}/lessons.md` under `## [REVIEW-NNN] — {ISO timestamp}`, capturing the pattern, why CodeRabbit caught it, and the root-cause class. Set `lessonsAppended: true` on the affected entries.
   - **Non-sprint branch** (`{{sprint}}` is `none`): there is no `lessons.md` or `FOLLOWUP.md`. Do NOT create either. Record the same pattern inline in the entry's `handoff.concerns`, and any deferral note in its `rationale`. Leave `lessonsAppended` `false`.

4. **Update the review.json at `{{review_json}}`** for EVERY entry in this batch:
   - `disposition`: `"fixed"`
   - `filesChanged`: array of files touched (excluding `review.json` and `lessons.md`)
   - `handoff`: `{ done: ["…"], decisions: ["…"], concerns: ["…"] }`
   - `lessonsAppended`: boolean
   - Leave `commitSha` as `null` — the orchestrator back-fills it post-commit.

   Also recompute `stats` and update `updatedAt`.

5. **Commit** using conventional format. One commit per batch.

   Single-entry: `fix(review): {summary} [{id}]`
   Multi-entry: `fix(review): {batch_label} [{id1}, {id2}, ...]`

   Include `Addresses CodeRabbit comments: {comma-separated #githubCommentId list}` in the body for multi-entry commits. The `[{id}]` tags in the subject are the canonical link — the orchestrator greps for them.

6. **Emit** `<promise>DONE</promise>` on success, `<promise>BLOCKED</promise>` on failure.

## Hard rules

- **One commit per batch.** All entries in the batch land in a single commit.
- **Never modify unrelated files.** If you spot another issue while fixing, leave it alone.
- **Do not rerun `intake` or touch other entries' dispositions.** You own exactly the entries in this batch.
- **Do not post GitHub replies.** That's a later phase.
- **No type-escape hatches** (casts, `any`, non-null assertions, suppressions) beyond what the project's rule docs allow. Fix types at the root.
- **Redact secrets** in any new log statements.
- **Prefer `counterFixPlan`** from `secondOpinion` when codex disagrees with the original `fixPlan`.
