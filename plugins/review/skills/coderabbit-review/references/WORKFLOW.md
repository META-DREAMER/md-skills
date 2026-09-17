# WORKFLOW — Phase Runbook

Per-phase detail for the orchestrator. Each phase is idempotent: it reads `review.json`, does its work, writes back. If a phase has nothing to do (all entries already past its gate), it prints a one-liner and returns.

## Shared Setup

At the start of every phase: read `<repo>/.agents/config.yaml` for `sprint_root` (default `sprints/`) and `worklog_root` (default `worklogs/`), detect the PR via `gh pr view --json number`, then resolve the `review.json` location by whether the branch maps to a sprint:

- **Sprint branch** — branch matches `sprint/{NNN}-{slug}` or legacy `ralph/{NNN}-{slug}` → `sprint="${NNN}-${slug}"`, `review_json="${sprint_root}/${sprint}/review.json"`.
- **Non-sprint branch** — branch matches neither pattern (`fix/...`, `feat/...`, any arbitrary name) → `sprint=null`, `slug="${branch#*/}"` (branch minus its type prefix), `review_json="${worklog_root}/${slug}/review.json"` (run `mkdir -p "${worklog_root}/${slug}"` first). If the branch already has a worklog from the `orchestrate` skill, this is that same folder — reuse it, never create a second one.

The rule is purely structural: the branch maps to a sprint or it doesn't. Do NOT fall back to the most recent sprint from any loop-state cursor — that pollutes an unrelated sprint's durable record and drifts as work advances. If `--resume` was passed or `review_json` exists, load it; else init empty.

### Sync on every run

Every invocation re-fetches PR threads via GraphQL, ingests new `githubCommentId`s not yet in `review.json`, and triages any entries still `disposition: "pending"`. Only after sync does the orchestrator proceed to the requested phase. Pass `--no-sync` to skip.

### Resume semantics

- Every phase checks `disposition` / `replyPosted` to skip completed work
- A crashed fix loop resumes at the next batch whose id tags are absent from `git log --since='{generatedAt}' --grep='{id}' -E` (authoritative — don't trust `commitSha` alone since it's a cache; scope by `generatedAt` and match the bare id token, since `[{id}]` collides across reviews and won't match a multi-id `[a, b]` subject — see SHA back-fill)
- On resume, back-fill any `fixed` entry whose `commitSha` is `null` or no longer resolves
- Re-running `intake` only adds new `githubCommentId`s, never overwrites existing dispositions

---

## Phase 1 — intake

**Goal:** Populate `review.json` with one entry per active CodeRabbit finding on the PR.

CodeRabbit surfaces findings in two places. Intake must harvest both — the first is usually just the top few items, the bulk lives in the second:

1. **Inline review threads** — GraphQL `reviewThreads`. Keep unresolved threads authored by `coderabbitai`; ingest only the top-level comment. Preserve `isOutdated: true` threads (triage decides if the concern still applies).
2. **Review-body sections** — `pulls/{pr}/reviews` bodies from `coderabbitai[bot]` contain collapsible `<details>` blocks grouping additional findings (commonly labelled `🔴 Critical`, `🟠 Major`, `🟡 Minor`, `🟣 Nitpick`, `🟢 Duplicate`). Each inner `<details>` is one finding whose summary line encodes `path` and line range. Parse them out.

Per entry, record `source` (`"thread"` or a `"review-{severity}"` variant), `path`, `lineRange`, stripped `body`, plus routing metadata: thread entries carry `githubCommentId`; review-body entries carry `reviewId` + `anchorUrl` and leave `githubCommentId` null (they aren't individually reply-addressable). Default severity from the section label; let a parsed badge line override it. Assign `review-NNN` ids deterministically across both sources.

### Body stripping

Strip CodeRabbit boilerplate: HTML comments, nested `<details>` for AI prompts / analysis chains / suggested diffs, severity badges (`_..._ | _..._`), "Also applies to" suffixes, orphaned tags. Collapse excess whitespace. Use judgment — the goal is a readable concern, not a perfect regex.

### Idempotency

Re-running intake must not duplicate entries. Dedup threads by `githubCommentId`; dedup review-body items by a stable key combining `reviewId`, `path`, `lineRange`, and a hash of the title. If a previously-ingested thread is gone (resolved on GitHub), mark `skipped` — don't delete. If a review-body item is gone from the latest review, leave the entry alone but set `stale: true` on the entry so triage can re-verify.

### Output

Print a one-line summary of what was kept vs. filtered across both sources. Write `review.json`. Commit as `chore(review): intake — {N} CodeRabbit findings from PR #{pr}`. Skip commit on `--resume` if nothing new was ingested.

---

## Phase 2 — triage

**Goal:** Fill in `disposition`, `confidence`, `rationale`, `fixPlan`, `needsSecondOpinion`, `severity`, `category` for every `pending` entry.

### Detect codex plugin

Detect via `CLAUDE_PLUGIN_ROOT` env var or plugin cache dir `~/.claude/plugins/cache/openai-codex/codex/*/`. Print detection status. If installed, kick off `node "{codex_companion}" adversarial-review --background` via `Bash` with `run_in_background: true` for a whole-branch second opinion surfaced during `approve`.

Apply a 5-minute timeout on the background process. If it fails to start, times out, or exits non-zero, log a warning (`codex second-opinion skipped: {reason}`) and continue the pipeline without codex data for flagged items — the approve phase proceeds based on triage alone. The approve phase detects completion by checking the background process exit status (or a sentinel output file) before consuming opinions; if not complete, skip codex opinions and note it in the approve summary.

### Group comments by topic

Cluster pending comments into coherent buckets by shared theme (e.g. "auth verification flow", "error handling around async DB calls"). A bucket may span multiple files.

Rules:
- Cap at **10 buckets**. Fewer is fine if comments cluster tightly.
- Genuinely unrelated singletons go in a `misc` catch-all.
- If total pending ≤ 10, use one bucket per comment or small groups.

**Guard:** if no pending comments, print `triage: no pending comments` and return early.

For each bucket, spawn one `Agent` with `subagent_type: Explore` (read-only). Send all spawns in a single message for true parallelism. Fill `assets/triage-brief.md` with `{{sprint}}` (the sprint slug, or `none` on a non-sprint branch), `{{path}}` (topic label), `{{comments_json}}`, and the config-derived `{{state_dir}}` (the resolved sprint or worklog folder), `{{rule_docs}}`, `{{docs_root}}`.

### Merge subagent results

Apply each subagent's returned patches to `review.json.comments` by `id`. Recompute `stats` and `updatedAt`. Per SCHEMA.md, `needsSecondOpinion` is optional and defaults to `false` when absent. A subagent may omit it, or include it as `true`. If a subagent returns `needsSecondOpinion: false` explicitly, strip it before merging to maintain schema compliance.

Commit as `chore(review): triage — {valid} valid / {invalid} invalid / {deferred} deferred / {fixed} fixed`.

### Codex second opinion (required when installed)

For entries flagged `needsSecondOpinion: true`, batch into up to 3 parallel `Agent` calls with `subagent_type: "codex:codex-rescue"`. Each returns `{ id, agree, rationale, counterFixPlan? }` per item. Merge into `review.json.comments[id].secondOpinion` as `{ source: "codex", agree, rationale, counterFixPlan? }`.

**Frame codex as devil's advocate, not validator.** The prompt must instruct codex to *argue the issue is FAKE* — find any reason CodeRabbit is wrong, the proposed fix is unnecessary, the concern is already handled elsewhere, or the cure is worse than the disease. Only return `agree: true` when codex cannot construct a credible refutation. Asking "is this real?" already concedes the issue exists; asking "prove this is fake" forces independent investigation.

Required prompt skeleton for each codex agent:

> Independent adversarial review on the following finding(s). Your job is to argue the issue is FAKE. Read the cited code yourself and look for: (a) the concern is already prevented elsewhere (upstream validation, type system, schema, framework guarantee); (b) the proposed fix would itself violate one of the project's rule docs (adds unnecessary comments / defensive validation / logging / wrappers); (c) CodeRabbit misread the code or cites a non-issue; (d) the cure is worse than the disease (more regression risk than the failure mode warrants). Only return `agree: true` when you cannot construct a credible refutation after honest investigation. Return `agree: false` with `counterFixPlan` if you have a materially better approach, or omit `counterFixPlan` if the right answer is "do nothing."

A codex disagreement rate below ~25% on flagged items is itself a signal — either triage is over-flagging easy items or codex is being too agreeable. Surface this in the approve narration.

Commit separately as `chore(review): codex second opinions — {N} items`.

---

## Phase 3 — approve

**Goal:** Get user approval on the triage before any code changes.

Print a summary table:

```text
review-001  src/api/auth/session.ts             valid     high     error-handling
review-002  src/api/middleware/...              invalid   high     lint          [reject reply queued]
...

Stats: 32 valid · 8 invalid · 6 deferred · 2 need your review
```

Use `AskUserQuestion` when at least one of:
- Any entry has `confidence: low`
- Codex disagrees (`secondOpinion.agree == false`)
- User passed `--interactive`

Questions surface only ambiguous items; clear-cut dispositions don't interrupt. Allow free-form user override ("flip review-005 to valid", "skip review-012"). Parse and update.

**Exit:** user types `approve` or orchestrator auto-approves if no ambiguities remain.

Commit overrides only if the user actually changed a disposition: `chore(review): approve — {N} overrides`.

---

## Phase 4 — fix

**Goal:** Implement every `valid` entry, one commit per batch.

### Ordering

Sort valid entries by package, severity (critical first), id.

### Batch formation

Group ordered entries into **batches** by file proximity and logical area. Target ~8–12 batches. A batch should touch files in the same package or tightly related cluster.

Rules:
- A batch MUST NOT span more than 2 packages.
- Large or risky entries get their own batch.
- Tiny unrelated remainders go in a `misc` catch-all.

### Subagent delegation (MANDATORY)

**The orchestrator MUST NOT apply fixes itself.** For each batch, spawn one `Agent` in foreground (batches are serial — later may depend on earlier compiling):

```typescript
Agent({
  description: "Fix batch: {batch_label}",
  mode: "auto",
  prompt: /* fill assets/fix-brief.md with {{sprint}} (sprint slug, or `none` on a non-sprint branch), {{review_json}} and {{state_dir}} (resolved in Shared Setup), {{rule_docs}}, {{docs_root}}, and {{comment_entries_json}} */
})
```

If a subagent reports `<promise>BLOCKED</promise>`, mark all entries in that batch as `blocked`, narrate the failure, continue to the next batch.

### Lessons and deferrals off a sprint branch

`<sprint_root>/{sprint}/lessons.md` and `<sprint_root>/{sprint}/FOLLOWUP.md` exist only on a sprint branch. On a non-sprint branch (sprint is null), the fix and triage subagents SKIP those writes entirely and instead record any durable lesson or deferral note inline in the entry: a reusable pattern goes in `handoff.concerns`, and a deferral note goes in `rationale`. The fix-brief and triage-brief carry this instruction via `{{sprint}}` being `none`.

At the end of the run the orchestrator rolls those up into `<worklog_root>/{slug}/handoff.md` — `## Lessons` and `## Follow-ups / open`, per the `orchestrate` skill — so the branch has one place a later reader looks instead of a ledger they have to mine. Subagents never append to it mid-run: concurrent markdown appends clobber, which is why the inline entry fields are the write path and the roll-up is a single authored pass. A lesson that generalizes past this branch graduates into the project's knowledge docs (the `rule_docs` surface) in that same commit.

### Post-batch verification

After each subagent returns, re-read `review.json` and verify every entry in the batch has a terminal disposition (`fixed` or `blocked`). If any entry is still `valid` (subagent committed code but didn't update `review.json`, or returned early with type errors), **re-dispatch the same batch** to a new subagent with a prompt that says: "The previous subagent left this batch incomplete — fix any remaining type errors, run the quality gate, update review.json, and commit." If the second attempt also fails, mark the entries `blocked` and move on. **The orchestrator MUST NOT apply fixes or clean up type errors itself.**

### SHA back-fill

After each batch commit, back-fill `commitSha` on all entries. **`[{id}]` tags are NOT globally unique** — a prior PR's review in shared git history may have reused `review-003`, and non-sprint reviews always restart numbering at `review-001`; a naive `git log --grep='\[review-003\]'` will match the wrong commit. A multi-id subject like `[review-002, review-003]` also won't match `\[review-003\]` (the id is mid-group). So **scope the window and loosen the match**: for each id run `git log --since='{generatedAt}' --grep='{id}' -E --format='%h' -n 1` — window bounded by this review's own `generatedAt` (from `review.json`), and the bare id token matches anywhere inside a `[a, b]` group, newest first. Failure modes: (a) zero matches → log a warning, leave `commitSha` null, and include the entry id in the final orchestrator summary under "missing SHAs"; (b) multiple matches → use the most recent; (c) the `[{id}]` tag in the commit subject is canonical per SCHEMA.md, so a null `commitSha` is NOT a failed fix — it only means the cache is stale. If the fix loop ends with any null SHAs that should be resolvable, make one final `chore(review): backfill commit shas` commit that re-derives them.

---

## Phase 5 — reply

**Goal:** Push the branch, then post batched threaded replies for every `invalid` / `deferred` / `blocked` entry where `replyPosted != true`.

### Push first

Before posting anything, `git push` the branch (confirm with the user if the pipeline hasn't already been told to push). Replies cite fix commit SHAs and the corrected code — posting before pushing leaves CodeRabbit and human readers with references to commits GitHub can't resolve, and CodeRabbit's re-review of the pushed commits should land alongside the replies, not after them.

### Preview

Print all drafted replies in chat. Ask one confirmation: `post all | revise | cancel`.

### Post

- **Thread entries** — post threaded replies via `gh api repos/{owner}/{repo}/pulls/{pr}/comments/{githubCommentId}/replies`. Body built from `assets/reply-template.md`.
- **Review-body entries** — not individually reply-addressable. Group by parent `reviewId` and post one consolidated PR comment per review (`issues/{pr}/comments`), listing each entry's path, line range, disposition, and rationale, and linking back to `anchorUrl`.

Set `replyPosted: true` after each successful post. Commit as `chore(review): mark replies posted — {N} entries`.
