---
name: coderabbit-review
description: "Triage and fix CodeRabbit PR review comments via parallel topic-based triage and serial fix phases, committing per phase. Cross-checks flagged items with codex when installed. Use when addressing CodeRabbit review on a PR, processing review.json, or running /coderabbit-review. Triggers: address pr comments, coderabbit review, review comments, fix pr review, triage pr review."
---

# CodeRabbit Review Orchestrator

Triage and fix a CodeRabbit review on a PR: parallel triage, a user approval gate, serial fixes, batched replies. The orchestrator coordinates and narrates; subagents write the code.

## Repo facts

**Repo facts.** Resolve each fact below in order: its `.agents/config.yaml` key; what the project's `CLAUDE.md`, `AGENTS.md` or README names; the conventional places listed. Relative paths resolve from the repo root. A fact you only read and cannot find is skipped with a note; one you must write is created at the default shown. Never guess a command. Say what you resolved, and from where, in your first status line.

| Fact | Key | Look for | If none |
| --- | --- | --- | --- |
| Product and technical docs | `docs_root` | the docs `CLAUDE.md` or README points to; `docs/` | work from the code and note the gap |
| Sprint folders | `sprint_root` | an existing `sprints/` | create `sprints/` |
| Worklogs | `worklog_root` | an existing `worklogs/` | create `worklogs/` |
| Rule docs | `rule_docs` | root and package `CLAUDE.md` / `AGENTS.md`, and the docs they route to | root `CLAUDE.md` or `AGENTS.md` |
| Package tests | `package_test_cmd` | `CLAUDE.md`, the package's `test` script, the CI workflow | the package's `test` script |
| Package typecheck | `package_typecheck_cmd` | same places | the package's `typecheck` script, else skip |

**State location.** A branch matching `sprint/{NNN}-{slug}` (or legacy `ralph/{NNN}-{slug}`) keeps state at `<sprint_root>/{NNN}-{slug}/review.json`. Any other branch uses `<worklog_root>/{slug}/review.json`, where `{slug}` is the branch minus its type prefix: the `orchestrate` skill's folder. Details: WORKFLOW.md, Shared Setup.

## Invocation

```bash
/coderabbit-review                  # full pipeline with approval gates
/coderabbit-review --resume         # continue from existing review.json
/coderabbit-review --phase=triage   # one phase: intake|triage|approve|fix|reply
/coderabbit-review --pr=18          # override the detected PR (gh pr view --json number)
```

## Phases

1. **intake**: fetch PR comments via GraphQL and write a skeletal `review.json`.
2. **triage**: cluster pending comments into topic buckets and triage them with parallel read-only subagents. Flagged items get a codex second opinion via `codex:codex-rescue` when installed; if codex fails, log it and continue on triage alone. **A comment cluster that exposes a vocabulary inconsistency or a contradiction with the spec docs is load-bearing**: route the fix and the doc patch under `docs_root` in the same batch, so docs and code never disagree.
3. **approve**: narrate findings; ask the user only about low-confidence or codex-disputed items.
4. **fix**: batch valid entries by file and area; one subagent per batch implements, tests and commits. The orchestrator never applies fixes itself.
5. **reply**: push the branch (with user confirmation), then batch-post threaded replies for `invalid`, `deferred` and `blocked` entries. Replies cite fix commits, so the push lands first.

Each phase is idempotent and commits once when something changed; fix commits are per batch.

## Files

- [WORKFLOW.md](references/WORKFLOW.md): per-phase runbook.
- [SCHEMA.md](references/SCHEMA.md): `review.json` shape and disposition lifecycle.
- [triage-brief.md](assets/triage-brief.md), [fix-brief.md](assets/fix-brief.md): subagent prompts.
- [reply-template.md](assets/reply-template.md): GitHub reply body.

## Narration

As each triage subagent completes, print comment id, path, disposition, confidence and rationale. As each fix batch commits, print entry ids, summary, files changed and whether a lesson was captured. Blocked batches get a short error excerpt.

Reusable patterns go to the sprint's `lessons.md` on a sprint branch. Off one, they go inline in the entry (`handoff.concerns`; deferral notes in `rationale`), and the orchestrator rolls them up at close-out into `<worklog_root>/{slug}/handoff.md` under `## Lessons` and `## Follow-ups / open`. Anything that generalizes past the branch moves to the project's rule docs.

## Exit criteria

- Every entry has a terminal disposition: `fixed`, `invalid`, `deferred`, `blocked` or `skipped`.
- Every `fixed` entry has a commit whose subject contains its id and a back-filled `commitSha`, with the gate green.
- Every `invalid`, `deferred` and `blocked` entry has `replyPosted: true`.
- Off a sprint branch, surviving lessons and deferrals are rolled up into the worklog's `handoff.md`.
- The branch is pushed and the orchestrator prints a summary table.
