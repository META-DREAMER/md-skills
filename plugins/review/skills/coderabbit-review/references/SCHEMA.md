# review.json schema

**Location:** `<sprint_root>/{sprint}/review.json` on a sprint branch, `<worklog_root>/{slug}/review.json` otherwise. Resolution rule: WORKFLOW.md, Shared setup.

## Shape

```json
{
  "sprint": "030-api-auth",
  "pr": 18,
  "branch": "sprint/030-api-auth",
  "generatedAt": "2026-04-15T12:00:00Z",
  "updatedAt": "2026-04-15T12:34:00Z",
  "stats": { "total": 48, "pending": 0, "valid": 0, "invalid": 0, "deferred": 0, "fixed": 0, "blocked": 0, "skipped": 0 },
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

A non-sprint review is identical except `sprint: null` and `branch` holds the real branch name.

**Optional fields are omitted until set**, and consumers treat a missing field as its zero value (`false`, `null`, `[]`, `{}`).

## Top-level fields

| Field | Type | Notes |
|-------|------|-------|
| `sprint` | string \| null | Sprint slug; `null` marks a non-sprint review |
| `pr` | number | GitHub PR number |
| `branch` | string | The PR's branch |
| `generatedAt` | ISO 8601 | Set at intake; bounds the SHA back-fill window |
| `updatedAt` | ISO 8601 | Updated on every write |
| `stats` | object | `total` plus counts by disposition, recomputed on every write |
| `comments` | array | One entry per finding |

## Intake fields

| Field | Type | Notes |
|-------|------|-------|
| `id` | string | `review-NNN`, deterministic across both sources |
| `githubCommentId` | number \| null | GraphQL `comments.nodes[0].databaseId`, the REST id replies need. Null for review-body entries |
| `source` | string | Omitted for threads. Review-body tokens `review-low`, `review-medium`, `review-high`, `review-critical` set the matching `severity`; `review-nitpick` and `review-duplicate` leave it for triage |
| `reviewId`, `anchorUrl` | | Review-body entries, so replies can point at the parent review |
| `dedupKey` | string | Review-body intake key (threads dedup on `githubCommentId`) |
| `stale` | boolean | Review-body item missing from the latest review |
| `path` | string | File path |
| `lineRange` | string | e.g. `"56-78"`, from `line` / `startLine` / `originalLine` |
| `isOutdated` | boolean | GitHub marked the thread outdated; refreshed on every sync |
| `summary` | string | First `**bold line**` of the stripped body, else the first line trimmed to 120 chars |
| `body` | string | Stripped comment body |

## Triage fields

| Field | Type | Notes |
|-------|------|-------|
| `disposition` | `pending` \| `valid` \| `invalid` \| `deferred` \| `fixed` \| `blocked` \| `skipped` | Lifecycle below |
| `confidence` | `low` \| `medium` \| `high` | Triage's confidence in its own call |
| `severity` | `low` \| `medium` \| `high` \| `critical` | Placeholder at intake, set by triage |
| `category` | string | Short slug (`error-handling`, `security`, `types`, `docs`, `ci`, `tests`) |
| `rationale` | string | 1 to 3 sentences |
| `fixPlan` | string | Concrete fix; omitted for `invalid` and `deferred` |
| `needsSecondOpinion` | boolean | Only ever present as `true` |
| `secondOpinion` | `{ source: "codex", agree, rationale, counterFixPlan? }` | Codex's verdict on a flagged item |

## Fix fields

| Field | Type | Notes |
|-------|------|-------|
| `handoff` | `{ done[], decisions[], concerns[] }` | Same shape as the plan-of-record handoff. Off a sprint branch, `concerns` also carries durable lessons |
| `filesChanged` | string[] | Files the fix touched |
| `commitSha` | string \| null | 7-char SHA back-filled by the orchestrator |
| `lessonsAppended` | boolean | The fix appended to the sprint's `lessons.md`; never set off a sprint branch |
| `replyPosted` | boolean | Set by the reply phase |

**`commitSha` is a cache, not state.** The `[{id}]` tag in the commit subject is the authoritative link. Re-derive a missing or dead SHA per WORKFLOW.md, SHA back-fill.

## Disposition lifecycle

```text
pending ──triage──▶ valid     ──fix──▶  fixed     (new fix commit)
                 ├─ fixed                 (already addressed in current code)
                 ├─ invalid   ──reply─▶  replyPosted
                 ├─ deferred  ──reply─▶  replyPosted
                 └─ skipped               (user opt-out, or thread resolved on GitHub)
valid ──fix──▶ blocked                    (quality gate failed twice)
```

- `fixed` from triage: the current code already addresses the concern, common for outdated threads after a force-push. `commitSha` is the pre-existing commit that fixed it (`git log -L` or `git blame`).
- `invalid`: CodeRabbit is wrong or the issue doesn't exist.
- `deferred`: CodeRabbit is right but the fix belongs later. Record it in the sprint's `FOLLOWUP.md`, or in `rationale` off a sprint branch.
- `skipped`: the user opted out during approve, or the thread was resolved on GitHub.
- `blocked`: the fix failed the gate twice; needs a human.
