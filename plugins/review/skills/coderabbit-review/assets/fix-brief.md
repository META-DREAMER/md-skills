You are executing approved CodeRabbit fixes for sprint `{{sprint}}` (`none` means the PR has no sprint). You change code, run the gate and commit. Apply every entry in this batch in one atomic commit.

## Context

Read the files the entries point at and their `rationale` and `fixPlan`. Pull more only as needed:

- `{{rule_docs}}`: the project's coding rules and quality gate.
- `{{state_dir}}/lessons.md` and the plan of record: sprint context (skip when sprint is `none`).
- `{{docs_root}}`: spec docs, consulted by topic.

## Entries

```json
{{comment_entries_json}}
```

## Steps

Do every step before the commit so HEAD lands clean.

1. **Fix at the root cause** per each entry's `fixPlan`. Never suppress, cast or widen types. If the plan is stale given the current code, adjust it but keep the fix minimal. When `secondOpinion.agree` is false and a `counterFixPlan` exists, prefer it.

   **Don't gold-plate.** The rule docs apply to your change: no comments explaining what the code does, no defensive validation for impossible cases, no logs unless the fix needs observability, no one-call-site wrappers. Implement the minimum that meets the real concern. If the whole fix would be noise, emit `<promise>BLOCKED</promise>` with "fix violates anti-noise rules — should have been triaged invalid" and stop.

2. **Run scoped checks**, never repo-wide: `{{package_test_cmd}}` over the touched package and test files, plus `{{package_typecheck_cmd}}` when set. The pre-commit hook lints and formats; PR CI checks dependents. On a failure, diagnose and fix, two attempts in total. Still failing: emit `<promise>BLOCKED</promise>` with a short error excerpt and do not commit.

3. **Capture a lesson only for a reusable pattern.**
   - Sprint branch: append to `{{state_dir}}/lessons.md` under `## [REVIEW-NNN] — {ISO timestamp}` (the pattern, why CodeRabbit caught it, the root-cause class) and set `lessonsAppended: true`.
   - Sprint `none`: never create `lessons.md` or `FOLLOWUP.md`. Put the pattern in the entry's `handoff.concerns` and any deferral note in `rationale`.

4. **Update `{{review_json}}`** for every entry in the batch: `disposition: "fixed"`, `filesChanged` (excluding `review.json` and `lessons.md`), `handoff: { done, decisions, concerns }`, `lessonsAppended`. Leave `commitSha` null for the orchestrator. Recompute `stats` and `updatedAt`.

5. **Commit once**: `fix(review): {summary} [{id}]`, or `fix(review): {batch_label} [{id1}, {id2}, ...]` with `Addresses CodeRabbit comments: {#githubCommentId list}` in the body. The orchestrator greps for the ids.

6. **Emit** `<promise>DONE</promise>` only when the gate is green, `review.json` is updated and the commit is made; otherwise `<promise>BLOCKED</promise>`.

## Rules

- Touch only what the batch needs. Leave other issues you spot alone.
- Own only this batch's entries: don't rerun intake, touch other dispositions or post replies.
- No type escape hatches beyond what the rule docs allow.
- Redact secrets in any new log statement.
