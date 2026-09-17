# review.json Schema

**Location:** `<sprint_root>/{sprint}/review.json` on a sprint branch; `<worklog_root>/{slug}/review.json` on a non-sprint branch (`sprint: null`). See WORKFLOW.md → Shared Setup for the resolution rule.

## Shape

```json
{
  "sprint": "030-api-auth",
  "pr": 18,
  "branch": "sprint/030-api-auth",
  "generatedAt": "2026-04-15T12:00:00Z",
  "updatedAt": "2026-04-15T12:34:00Z",
  "stats": {
    "total": 48,
    "pending": 0,
    "valid": 0,
    "invalid": 0,
    "deferred": 0,
    "fixed": 0,
    "blocked": 0,
    "skipped": 0
  },
  "comments": [
    {
      "id": "review-001",
      "githubCommentId": 3088783114,
      "path": "src/api/auth/session.ts",
      "lineRange": "56-78",
      "isOutdated": false,
      "severity": "high",
      "category": "error-handling",
      "summary": "Wrap SIWE parse/verify in try/catch",
      "body": "**Wrap SIWE parse/verify in try/catch.**\n\n`parseSiweMessage()` and `verifySiweMessage()` throw typed exceptions on invalid input. Without try-catch, these surface as unhandled 500s.",

      "disposition": "pending",
      "confidence": "high",
      "rationale": "",
      "fixPlan": ""
    }
  ]
}
```

The example above is a sprint review. A non-sprint review (file at `worklogs/{slug}/review.json`) is identical except `sprint: null` and `branch` holds the real branch name (e.g. `"fix/wallet-funding-ux"`).

Fields that appear only when populated (omit when default):
- `source` — omit for inline threads (default `"thread"`); for review-body items use one of the tokens below. Allowed values: `"thread"`, `"review-low"`, `"review-medium"`, `"review-high"`, `"review-critical"`, plus the workflow-only labels `"review-nitpick"` and `"review-duplicate"`. Mapping from `review-*` token to canonical `severity`:

  | `source` token       | `severity`        |
  |----------------------|-------------------|
  | `review-low`         | `low`             |
  | `review-medium`      | `medium`          |
  | `review-high`        | `high`            |
  | `review-critical`    | `critical`        |
  | `review-nitpick`     | (not set — triage assigns) |
  | `review-duplicate`   | (not set — triage assigns) |

  `review-nitpick` and `review-duplicate` are workflow-only labels that come from CodeRabbit section headers and do not set `severity` automatically; triage must assign one.
- `reviewId` / `anchorUrl` — set on review-body entries so replies can point back to the parent review
- `dedupKey` — stable intake key for review-body entries (threads dedup on `githubCommentId`)
- `stale` — intake flag when a previously-ingested review-body item is no longer in the latest review
- `needsSecondOpinion` — omit when `false`; include as `true` when triage flags it
- `secondOpinion` — omit when absent; set by codex during triage
- `handoff` — omit until fix phase populates `{ done, decisions, concerns }`
- `filesChanged` — omit until fix phase populates the array
- `commitSha` — omit until orchestrator back-fills post-commit
- `lessonsAppended` — omit until fix phase sets `true`
- `replyPosted` — omit until reply phase sets `true`

Consumers must treat missing optional fields as their zero value (`false`, `null`, `[]`, `{}`).

## Field Definitions

### Top-level

| Field | Type | Notes |
|-------|------|-------|
| `sprint` | string \| null | Sprint slug, e.g. `030-api-auth`; `null` for a non-sprint review (identified solely by this — `branch` holds the real branch name) |
| `pr` | number | GitHub PR number |
| `branch` | string | Branch the PR targets (`sprint/{sprint}` or legacy `ralph/{sprint}` on a sprint branch; any other name when `sprint` is `null`) |
| `generatedAt` | ISO 8601 | Set during `intake` |
| `updatedAt` | ISO 8601 | Updated on every `review.json` write |
| `stats` | object | Counts by disposition; recomputed on every save |
| `comments` | array | One entry per CodeRabbit inline comment |

### Per-comment (ingested fields — set during intake)

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | `review-NNN`, stable ordering by GitHub comment id |
| `githubCommentId` | number | From GraphQL `comments.nodes[0].databaseId` — the REST numeric id needed for posting replies |
| `path` | string | File path from the review comment |
| `lineRange` | string | e.g. `"56-78"`; derived from `line` / `startLine` / `originalLine` |
| `isOutdated` | boolean | GitHub marked the thread outdated (code changed under it). Triage must verify whether the concern is already addressed. Refreshed on every sync. |
| `severity` | `"low"` \| `"medium"` \| `"high"` \| `"critical"` | Assigned during triage, placeholder at intake |
| `category` | string | Short slug (e.g. `error-handling`, `security`, `types`, `docs`, `ci`, `tests`) |
| `summary` | string | First `**bold line**` from the stripped body. Fallback: first non-empty line, trimmed to 120 chars. |
| `body` | string | Stripped CodeRabbit comment body (boilerplate removed during intake — see WORKFLOW.md Phase 1) |

### Per-comment (triage-written fields — set during triage)

| Field | Type | Notes |
|-------|------|-------|
| `disposition` | `"pending"` \| `"valid"` \| `"invalid"` \| `"deferred"` \| `"fixed"` \| `"blocked"` \| `"skipped"` | Lifecycle below |
| `confidence` | `"low"` \| `"medium"` \| `"high"` | Triage agent's confidence in its own disposition |
| `needsSecondOpinion` | boolean | Triage flags `true` for auth/security/DeFi-safety/architectural items. **Omit when `false`** — consumers treat missing as `false`. |
| `rationale` | string | Why the triage agent chose this disposition (1–3 sentences) |
| `fixPlan` | string | Concrete approach for the fix. **Omit when empty** (i.e. for `invalid` or `deferred`). |

### Per-comment (codex-written field — set during triage if plugin is installed)

| Field | Type | Notes |
|-------|------|-------|
| `secondOpinion` | `{ source: "codex", agree: boolean, rationale: string, counterFixPlan?: string } \| null` | Set when codex weighs in on a `needsSecondOpinion: true` item |

### Per-comment (fix-written fields — set during fix phase, omit until populated)

All fix-phase fields are **omitted from the entry until the phase that writes them**. Consumers must treat missing values as their zero value (`[]`, `null`, `false`).

| Field | Type | Notes |
|-------|------|-------|
| `handoff` | `{ done: string[], decisions: string[], concerns: string[] }` | Same shape as the project's plan-of-record handoff block. Added by fix subagent. On a non-sprint branch this also absorbs durable lessons (in `concerns`) that would otherwise go to `lessons.md`. |
| `filesChanged` | string[] | Files touched by the fix commit. Added by fix subagent. |
| `commitSha` | string \| null | Short SHA (7 chars). Convenience cache — canonical link is the `[{id}]` tag in the commit subject. Back-filled by orchestrator post-commit. |
| `lessonsAppended` | boolean | `true` if the fix appended to `<sprint_root>/{sprint}/lessons.md`. Added by fix subagent. Always `false`/omitted on a non-sprint branch (no `lessons.md`; lesson recorded in `handoff.concerns` instead). |
| `replyPosted` | boolean | Set by `reply` phase. |

**`commitSha` is a cache, not state.** The commit subject's `[{id}]` tag is the authoritative link. If `commitSha` is missing, empty, or no longer resolves (e.g. after a squash/rebase), re-derive it with `git log --grep='\[review-NNN\]' --format='%h'` and write back.

## Disposition Lifecycle

```text
pending ──triage──▶ valid     ──fix──▶  fixed   (new fix commit)
                 ├─ fixed     (already-addressed: code on disk matches the concern)
                 ├─ invalid   ──reply─▶  (replyPosted: true)
                 ├─ deferred  ──reply─▶  (replyPosted: true)
                 └─ skipped   (user opt-out, or thread resolved on GitHub after ingest)

             fix may transition to:
             valid  ──▶  blocked   (quality gate failed twice)
```

- `fixed` (from triage) = triage agent verified the concern is already addressed in the current code (common for `isOutdated: true` items after a force-push). No new commit needed; set `commitSha` to the short SHA of the pre-existing commit that addressed it (look it up via `git log -L` or `git blame`). `replyPosted` still cycles through `reply` phase to close the loop with CodeRabbit.
- `invalid` = CodeRabbit is wrong or the issue doesn't exist
- `deferred` = CodeRabbit is right but fix belongs in a later sprint (capture in `<sprint_root>/{sprint}/lessons.md` as follow-up; on a non-sprint branch there is no `lessons.md` — capture the deferral note inline in the entry's `rationale`)
- `skipped` = user explicitly opted out during `approve`, OR thread was resolved on GitHub after ingest
- `blocked` = fix attempted, quality gate failed twice; needs human

## Stats

Recomputed on every `review.json` write. Counts by disposition plus `total`.
