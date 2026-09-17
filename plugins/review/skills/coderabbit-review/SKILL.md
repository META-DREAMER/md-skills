---
name: coderabbit-review
description: "Triage and fix CodeRabbit PR review comments via parallel topic-based triage and serial fix phases, committing per phase. Cross-checks flagged items with codex when installed. Use when addressing CodeRabbit review on a PR, processing review.json, or running /coderabbit-review. Triggers: address pr comments, coderabbit review, review comments, fix pr review, triage pr review."
---

# CodeRabbit Review Orchestrator

Run the full pipeline to triage and fix a CodeRabbit review on a PR: parallel triage, a user approval gate, serial fixes, and batched reply-posting. The orchestrator coordinates and narrates; subagents write the code.

## Configuration

Read `<repo>/.agents/config.yaml` when it exists. Used here:

| Key | Use |
|-----|-----|
| `sprint_root` / `worklog_root` | Where `review.json` and the close-out files live (see below) |
| `docs_root` | The project's spec docs, patched when a fix changes documented behaviour |
| `rule_docs` | The rules a fix must honour, and the rules codex argues a proposed fix would violate |
| `package_test_cmd`, `checkpoint_cmd` | The gate each fix subagent runs before committing |

Defaults when the file is absent: `sprints/`, `worklogs/`, `docs/`, every `CLAUDE.md` in the repo, and whatever test command the project's own docs name. Say in the run summary that you used defaults.

**State location, by convention:** a branch matching `sprint/{NNN}-{slug}` (or legacy `ralph/{NNN}-{slug}`) stores state at `<sprint_root>/{NNN}-{slug}/review.json`; any other branch stores it at `<worklog_root>/{slug}/review.json`, where `{slug}` is the branch minus its type prefix — the same folder the `orchestrate` skill uses. See WORKFLOW.md → Shared Setup.

## Invocation

```bash
/coderabbit-review                  # full pipeline with approval gates
/coderabbit-review --resume         # pick up from existing review.json state
/coderabbit-review --phase=triage   # run a single phase (intake|triage|approve|fix|reply)
/coderabbit-review --pr=18          # override auto-detected PR number
```

PR number and branch mapping are auto-detected. PR is `gh pr view --json number`. Override only if detection fails.

## Phases

1. **intake** — fetch PR comments via GraphQL, write skeletal `review.json`.
2. **triage** — cluster pending comments into topic buckets; spawn parallel read-only subagents; flagged items get a codex second opinion (when installed) via `codex:codex-rescue`. If the codex plugin is installed but fails (partial install, malformed output, timeout), log it and continue — the approve phase still works on triage alone. **When a comment cluster surfaces a vocabulary inconsistency or a contradiction with the project's spec docs, treat it as load-bearing**: route the fix and propose the doc patch under `docs_root` in the same batch. Never leave the docs and the code disagreeing.
3. **approve** — narrate findings; ask the user only about low-confidence or codex-disputed items.
4. **fix** — batch valid entries by file and area; spawn one subagent per batch to implement, test, and commit. The orchestrator **never applies fixes directly**.
5. **reply** — push the branch (with user confirmation), then batch-post threaded GitHub replies for `invalid` / `deferred` / `blocked` dispositions. Replies cite fix commits, so the push must land first.

Each phase is idempotent; re-running a completed phase is a no-op. Each phase commits, so the PR diff shows distinct, reviewable steps. Fix commits are per batch; other phases commit once. Skip the commit when nothing changed.

## References

- [WORKFLOW.md](references/WORKFLOW.md) — per-phase runbook
- [SCHEMA.md](references/SCHEMA.md) — `review.json` shape, disposition lifecycle, field definitions

## Subagent templates

- [triage-brief.md](assets/triage-brief.md) — fed to each parallel triage subagent
- [fix-brief.md](assets/fix-brief.md) — fed to the serial fix executor
- [reply-template.md](assets/reply-template.md) — GitHub threaded reply body

## Narration

As each triage subagent completes, print the comment id, path, disposition, confidence, and rationale to chat. As each fix batch commits, print the entry ids, summary, files changed, and whether a lesson was captured. Blocked batches get a short error excerpt. The user gets a running view without reading `review.json`.

When a fix surfaces a reusable pattern, it goes to the sprint's `lessons.md` on a sprint branch; off one it goes inline in the entry (`handoff.concerns`, deferral notes in `rationale`), and the orchestrator rolls both up at close-out into `<worklog_root>/{slug}/handoff.md` under `## Lessons` and `## Follow-ups / open`. Anything that generalizes past the branch graduates into the project's knowledge docs.

## Exit criteria

- Every entry in `review.json.comments` has a terminal disposition (`fixed` | `invalid` | `deferred` | `blocked` | `skipped`).
- Every `fixed` entry has a commit whose subject contains `[{id}]` and a back-filled `commitSha`, with the gate passing.
- Every `invalid` / `deferred` / `blocked` entry has `replyPosted: true`.
- Off a sprint branch, surviving lessons and deferrals are rolled up into the worklog's `handoff.md` — not left only in entry fields.
- The branch is pushed (the reply phase pushes before posting); the orchestrator prints a summary table.
