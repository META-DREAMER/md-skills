# WORKFLOW: phase runbook

Each phase reads `review.json`, does its work and writes back. With nothing to do, it prints a one-liner and returns.

## Shared setup

At the start of every phase, resolve `sprint_root` and `worklog_root` (SKILL.md, Repo facts), detect the PR with `gh pr view --json number`, and resolve `review.json`:

- **Sprint branch** (`sprint/{NNN}-{slug}` or legacy `ralph/{NNN}-{slug}`): `sprint="${NNN}-${slug}"`, `review_json="${sprint_root}/${sprint}/review.json"`.
- **Any other branch**: `sprint=null`, `slug="${branch#*/}"`, `review_json="${worklog_root}/${slug}/review.json"` (`mkdir -p` the folder first). If the `orchestrate` skill already made a worklog for the branch, reuse that folder.

**The rule is structural: the branch maps to a sprint or it doesn't.** Never fall back to the latest sprint from a loop-state cursor; that pollutes an unrelated sprint's record. If `--resume` was passed or `review_json` exists, load it; else start empty.

**Sync on every run.** Each invocation re-fetches PR threads, ingests new `githubCommentId`s and triages any `pending` entries before running the requested phase. `--no-sync` skips it.

**Resume.**

- Every phase checks `disposition` and `replyPosted` to skip completed work.
- A crashed fix loop resumes at the next batch whose ids are absent from `git log --since='{generatedAt}' --grep='{id}' -E`. Git is authoritative; `commitSha` is a cache (see SHA back-fill).
- Back-fill any `fixed` entry whose `commitSha` is null or no longer resolves.
- Re-running intake only adds new entries, never overwrites dispositions.

## Phase 1: intake

Populate `review.json` with one entry per active CodeRabbit finding. **CodeRabbit puts findings in two places, and the bulk is in the second:**

1. **Inline review threads**: GraphQL `reviewThreads`. Keep unresolved threads authored by `coderabbitai`, top-level comment only. Keep `isOutdated: true` threads; triage decides whether they still apply.
2. **Review-body sections**: `pulls/{pr}/reviews` bodies from `coderabbitai[bot]` hold collapsible `<details>` blocks grouping more findings (`🔴 Critical`, `🟠 Major`, `🟡 Minor`, `🟣 Nitpick`, `🟢 Duplicate`). Each inner `<details>` is one finding whose summary line encodes path and line range.

Per entry, record `source`, `path`, `lineRange` and the stripped `body`. Thread entries carry `githubCommentId`; review-body entries carry `reviewId` and `anchorUrl` with `githubCommentId` null, since they are not individually reply-addressable. Severity defaults from the section label; a parsed badge overrides it. Assign `review-NNN` ids deterministically across both sources.

**Strip boilerplate**: HTML comments, nested `<details>` for AI prompts, analysis chains and suggested diffs, severity badges, "Also applies to" suffixes, orphaned tags, excess whitespace. The goal is a readable concern.

**Idempotency.** Dedup threads by `githubCommentId` and review-body items by `dedupKey` (`reviewId`, `path`, `lineRange`, title hash). A previously ingested thread that is gone (resolved on GitHub) becomes `skipped`, never deleted. A review-body item missing from the latest review gets `stale: true` for triage to re-verify.

Print kept vs filtered counts per source. Commit `chore(review): intake — {N} CodeRabbit findings from PR #{pr}`, skipped on `--resume` when nothing new arrived.

## Phase 2: triage

Fill `disposition`, `confidence`, `rationale`, `fixPlan`, `needsSecondOpinion`, `severity` and `category` for every `pending` entry. With none pending, print `triage: no pending comments` and return.

**Codex whole-branch pass.** Detect the plugin via `CLAUDE_PLUGIN_ROOT` or `~/.claude/plugins/cache/openai-codex/codex/*/` and print the result. If installed, start `node "{codex_companion}" adversarial-review --background` in a `Bash` call with `run_in_background: true`, with a 5-minute timeout. If it fails, times out or exits non-zero, log `codex second-opinion skipped: {reason}` and continue. Approve checks the process exit status (or a sentinel file) before using its output, and notes when it was not ready.

**Bucket by topic.** Cluster pending comments by shared theme ("auth verification flow", "error handling around async DB calls"); a bucket may span files. At most 10 buckets; unrelated singletons go in `misc`. With 10 or fewer pending, one bucket per comment or small groups is fine.

Spawn one read-only `Agent` (`subagent_type: Explore`) per bucket, all in one message. Fill [triage-brief.md](../assets/triage-brief.md) with `{{sprint}}` (slug or `none`), `{{path}}` (topic label), `{{comments_json}}`, `{{state_dir}}` (the resolved sprint or worklog folder), `{{rule_docs}}` and `{{docs_root}}`.

**Merge** each subagent's patches into `review.json.comments` by `id`, recompute `stats` and `updatedAt`, and strip any explicit `needsSecondOpinion: false`. Commit `chore(review): triage — {valid} valid / {invalid} invalid / {deferred} deferred / {fixed} fixed`.

**Codex second opinion (required when installed).** Batch entries flagged `needsSecondOpinion: true` into up to 3 parallel `Agent` calls with `subagent_type: "codex:codex-rescue"`. Each returns `{ id, agree, rationale, counterFixPlan? }` per item, merged into `secondOpinion` as `{ source: "codex", ... }`.

**Frame codex as devil's advocate.** Asking "is this real?" concedes the issue exists; asking it to prove the issue fake forces independent investigation. Use this prompt:

> Independent adversarial review on the following finding(s). Your job is to argue the issue is FAKE. Read the cited code yourself and look for: (a) the concern is already prevented elsewhere (upstream validation, type system, schema, framework guarantee); (b) the proposed fix would itself violate one of the project's rule docs (adds unnecessary comments / defensive validation / logging / wrappers); (c) CodeRabbit misread the code or cites a non-issue; (d) the cure is worse than the disease (more regression risk than the failure mode warrants). Only return `agree: true` when you cannot construct a credible refutation after honest investigation. Return `agree: false` with `counterFixPlan` if you have a materially better approach, or omit `counterFixPlan` if the right answer is "do nothing."

A codex disagreement rate below about 25% on flagged items means triage is over-flagging or codex is too agreeable; say so in the approve narration. Commit `chore(review): codex second opinions — {N} items`.

## Phase 3: approve

Get user approval before any code changes. Print a summary table:

```text
review-001  src/api/auth/session.ts             valid     high     error-handling
review-002  src/api/middleware/...              invalid   high     lint          [reject reply queued]

Stats: 32 valid · 8 invalid · 6 deferred · 2 need your review
```

Use `AskUserQuestion` only when an entry has `confidence: low`, codex disagrees (`secondOpinion.agree == false`), or the user passed `--interactive`. Accept free-form overrides ("flip review-005 to valid", "skip review-012"). The phase exits when the user types `approve`, or automatically when nothing is ambiguous. Commit `chore(review): approve — {N} overrides` only when a disposition changed.

## Phase 4: fix

Implement every `valid` entry, one commit per batch.

**Batches.** Sort valid entries by package, severity (critical first), id. Group by file proximity and logical area into about 8 to 12 batches. A batch spans at most 2 packages; large or risky entries get their own; small remainders go in `misc`.

**Delegate every batch.** The orchestrator never applies fixes or cleans up type errors itself. Spawn one foreground `Agent` per batch, serially, since later batches may depend on earlier ones compiling:

```typescript
Agent({
  description: "Fix batch: {batch_label}",
  mode: "auto",
  prompt: /* assets/fix-brief.md filled with {{sprint}}, {{review_json}}, {{state_dir}}, {{rule_docs}}, {{docs_root}}, {{package_test_cmd}}, {{package_typecheck_cmd}}, {{comment_entries_json}} */
})
```

On `<promise>BLOCKED</promise>`, mark the batch's entries `blocked`, narrate, and continue.

**Post-batch check.** Re-read `review.json`. If any batch entry is still `valid`, re-dispatch the batch once to a fresh subagent: "The previous subagent left this batch incomplete — fix any remaining type errors, run the quality gate, update review.json, and commit." A second failure marks the entries `blocked`.

**Lessons off a sprint branch.** `lessons.md` and `FOLLOWUP.md` exist only in a sprint folder. With `sprint` null, subagents skip those writes and record a reusable pattern in the entry's `handoff.concerns` and a deferral note in its `rationale`. At the end of the run the orchestrator rolls them up into `<worklog_root>/{slug}/handoff.md` (`## Lessons`, `## Follow-ups / open`) in one authored pass; subagents never append to it, because concurrent markdown appends clobber.

**SHA back-fill.** After each batch commit, back-fill `commitSha`. **`[{id}]` tags are not globally unique**: an earlier review in shared history may have used `review-003`, non-sprint reviews restart at `review-001`, and `\[review-003\]` misses a multi-id subject like `[review-002, review-003]`. So per id run `git log --since='{generatedAt}' --grep='{id}' -E --format='%h' -n 1`. Zero matches: warn, leave `commitSha` null and list the id under "missing SHAs" in the summary. Multiple: use the newest. A null `commitSha` is a stale cache, not a failed fix. If resolvable SHAs are still null at the end, commit `chore(review): backfill commit shas`.

## Phase 5: reply

Push the branch, then reply to every `invalid`, `deferred` and `blocked` entry without `replyPosted`.

**Push first** (confirm with the user unless already told to push). Replies cite fix SHAs that GitHub must resolve, and CodeRabbit's re-review of the pushed commits should land with the replies.

Print every drafted reply and ask once: `post all | revise | cancel`.

- **Thread entries**: `gh api repos/{owner}/{repo}/pulls/{pr}/comments/{githubCommentId}/replies`, body from [reply-template.md](../assets/reply-template.md). Send the body as JSON with `--input`; `-f body=@file` posts the literal string `@file`.
- **Review-body entries**: group by `reviewId` and post one PR comment per review (`issues/{pr}/comments`) listing each entry's path, line range, disposition and rationale, linking `anchorUrl`.

Set `replyPosted: true` after each post. Commit `chore(review): mark replies posted — {N} entries`.
