---
name: ralph-plan
description: "Generate spec.json from a sprint PRD: research, a pair-design conversation on the real forks, then a plan of record. Use when preparing a sprint for autonomous execution."
---

# Ralph Plan

Turns `<sprint_root>/{sprint}/PRD.md` into an executable `spec.json`. The builder is as capable as you: the spec fixes **intent and evidence** (what each story delivers and how it is proven) and leaves how to the builder.

## Repo facts

**Repo facts.** Resolve each fact below in order: its `.agents/config.yaml` key; what the project's `CLAUDE.md`, `AGENTS.md` or README names; the conventional places listed. Relative paths resolve from the repo root. A fact you only read and cannot find is skipped with a note; one you must write is created at the default shown. Never guess a command. Say what you resolved, and from where, in your first status line.

| Fact | Key | Look for | If none |
| --- | --- | --- | --- |
| Product and technical docs | `docs_root` | the docs `CLAUDE.md` or README points to; `docs/` | work from the code and note the gap |
| Glossary | `glossary` | a glossary or terminology section the project names; `docs/**/glossary*.md` | read: skip. Write-back: create `docs/glossary.md` |
| Architecture decisions | `adr_log` | `docs/adr/`, `docs/decisions*`, `DECISIONS.md`, `ADR*.md`, an architecture-decisions doc | read: skip. Recording one: create `docs/decisions.md` |
| Roadmap | `roadmap` | a roadmap the project names; `docs/**/roadmap*` | skip |
| Sprint folders | `sprint_root` | an existing `sprints/` | create `sprints/` |
| Rule docs | `rule_docs` | root and package `CLAUDE.md` / `AGENTS.md`, and the docs they route to | root `CLAUDE.md` or `AGENTS.md` |

## Source of truth

The PRD is the plan of record. `docs_root` is the baseline it extends: read it for vocabulary and settled decisions, then plan the sprint as written. A reference or roadmap doc that disagrees with the PRD is stale; note it for the write-back story. Exception: decisions in `adr_log` and the safety invariants in `rule_docs` change only through the ADR gate with the user's ratification. If the code contradicts the PRD's premise, tell the user.

## Input

```
/ralph-plan 020
/ralph-plan 020-execution-backbone
```

Prefix → match `<sprint_root>/020-*`; no match → error and list the sprints. `spec.json` exists → **Regenerate** mode; only `PRD.md` → **Normal**.

## Phase 1: Research

Read before forming an opinion:

1. The PRD and every doc under its Context.
2. `glossary` and `adr_log`: the vocabulary and constraints for everything after.
3. The sprint's `lessons.md` if present, and for the touched areas earlier sprints' handoffs (`spec-json` skill), `lessons.md` and `FOLLOWUP.md`.
4. **Cross-sprint synthesis.** Scan adjacent PRDs' Risks and open-gap statements for seams this sprint is positioned to close; each becomes a story or a named deferral in `interviewSummary`, never silence. A planned sprint that executes before this one is part of the baseline: plan against the code as it will be, keeping integration with it high-level.
5. The code the sprint touches. Verify the PRD's "how it works today" claims; quote file and line where code disagrees. Delegate broad exploration.
6. Third-party signatures against the installed package's types, official docs, or Context7. Training data is stale for version-sensitive APIs.

Come out with: what the sprint delivers, the two or three real design forks, and what the PRD leaves undecided.

## Phase 2: Design conversation

Open with your understanding (deliverables, candidate approaches, tradeoffs), then design out loud and put each fork to the user as it arises. The user never first sees a decision in the finished spec.json.

**Any decision that shapes architecture, data model, integration posture, scope or proof bar is the user's to ratify**, with your recommendation and why attached. The ones that go wrong are the ones a planner finds obvious: whether an integration gets its own package, who advances long-running state (client polling vs a server-side scheduler), extend a shape vs reshape to the end-state (pre-launch: recommend reshape), wire a cross-sprint seam now vs a named deferral, how much live proof and spend the sprint buys.

- Every question carries your recommended answer.
- Don't ask what docs, code or the PRD answer. Answers from a `create-sprint` interview earlier in the session stand.
- Batch only independent forks; a decision that changes the next question is asked alone.
- Challenge terms against `glossary` and claims against code and `adr_log`. Probe with a concrete failure scenario where the answer changes the plan ("the job fires twice for the same record: whose story handles that?").
- A question that only yields speculation is an **integration spike story**.

## Phase 3: Plan of record

Draft the story breakdown from the ratified decisions. **Deletion test** each story: if deleting its deliverable makes no complexity reappear, fold it into a neighbour. Favour stories that produce deep modules; use the `rule_docs` version of this principle. The test scopes stories and helpers, not the package boundaries ratified in Phase 2.

Present the plan (stories with deps, checkpoints, risks). Resolve remaining forks one question at a time, recommendation first. Speculative leftovers become spikes or `interviewSummary.risks`.

**ADR gate, before writing spec.json.** A decision that is hard to reverse, surprising without context, and the result of a real trade-off goes into `adr_log` in that file's format, plus a short subsection in the owning spec doc if the why needs more than a line. Don't manufacture ADRs.

**Inline doc updates.** A term sharpened in the interview that outlives the sprint → propose the `glossary` patch in that turn. Sprint-local discoveries → `lessons.md` Planning Context.

## spec.json

Write `<sprint_root>/{sprint}/spec.json`:

```json
{
  "sprint": "{NNN}-{slug}",
  "branch": "sprint/{NNN}-{slug}",
  "generatedAt": "ISO-8601 timestamp",
  "mode": "normal | regenerate",
  "stories": [
    {
      "id": "{NNN}-{XX}",
      "title": "Short descriptive title",
      "description": "Intent and boundaries — what this story delivers, why, and what's out of scope",
      "deps": ["NNN-XX"],
      "acceptanceCriteria": [
        "Specific criterion naming its evidence",
        "Another criterion naming its evidence"
      ],
      "testFirst": false,
      "difficulty": "med",
      "review": false,
      "passes": false,
      "notes": "Optional. Non-obvious gotchas, key doc references, edge cases the builder can't derive."
    }
  ],
  "interviewSummary": {
    "scopeChanges": "Any changes made during interview",
    "risks": ["Risk 1", "Risk 2"],
    "decisions": ["Decision 1", "Decision 2"]
  }
}
```

Story IDs are `{NNN}-{XX}`: `076-01`, `076-15`.

### Intent, not instructions

`description` carries what and why: scope boundaries and the contract with neighbouring stories. `notes` carries only what the builder can't derive: gotchas, doc sections, edge cases. **Never step-by-step implementation**; the builder reads the `rule_docs`, package rules and neighbouring code itself. Exception: `low` stories get prescriptive notes.

Always pin cross-story contracts and interview-settled decisions, with the why. Leave internals intent-only unless the story is `low`. Late stories lean further toward intent, since plan-time specifics go stalest there.

### Sizing

A story is one coherent, independently provable unit of intent, worth one iteration's overhead (pick, build, prove, handoff, polish, commit). Prefer fewer, deeper stories. Split when a story mixes concerns proven differently (schema vs UI vs live validation) or bundles a protected-surface change with unrelated work.

### Ordering

**Integration spikes first, runtime validation last.** Between them, order by dependency and build an early **vertical slice**, one thin end-to-end path, so integration surprises surface early. `deps` point only at lower sequence numbers, no cycles, `[]` when independent.

### Acceptance criteria

Typecheck, lint, format and the story's own tests are enforced already; don't restate them. Each criterion states what is proven and by what: a named test, a command and expected output, a `browser-verify` check, a read from the live environment. A criterion naming a live surface is proven on that surface.

Bad: "Works correctly." Good: "Returns health factor 1.5 for a position with $1500 collateral and $1000 debt."

Typical evidence: pure logic and state machines → named tests covering the transitions; API endpoints → request/response tests; UI → `browser-verify`; deployment → deployed target plus captured response; spike → the answer recorded, with the fallback if it failed.

### testFirst

`true` for pure logic, state machines, crypto and signing round-trips, API endpoints, and bug fixes. `false` for schema and migrations, UI, config and infra, wiring, deployment.

### difficulty

Sets the builder's reasoning effort. Absent = `med`.

- `low`: obvious shape (wiring, config, mechanical UI). Runs at reduced effort, so write prescriptive notes (files, functions, shapes) with the why.
- `med`: standard effort, intent-only notes.
- `high`: judgment-heavy or novel (state reconcilers, protocol encoding, cross-service protocols, tricky migrations). The builder implements, then gets one adversarial review of the diff.

Protected-surface stories, as the `rule_docs` define them, are never `low`.

### review

`review: true` marks a **checkpoint**: after the story, the loop runs `story-review` over the cumulative diff since the last checkpoint. Place checkpoints where the accumulated diff is a coherent unit: after a cluster of risky stories, at layer boundaries (last backend story before UI), on any single big or risky story, and always on the final implementation story before runtime validation. The loop also self-triggers a checkpoint on protected-surface diffs, so the flag sets cadence, not safety.

### Runtime-proven sprints

If the PRD requires real-environment proof, the spec carries it as first-class stories (staging deploy and verification, live migration, live auth check), never as a vague criterion on an unrelated story. The final validation story writes `<sprint_root>/{sprint}/VALIDATION.md`, never repo-root files, through the repo's live-validation skill if it has one.

### Docs write-back

Sprints run ahead of `docs_root`, so decide at plan time whether this one leaves a doc wrong: a surface it reshapes, a status marker it flips, a decision the interview changed, a doc the PRD already flags as stale. If so, the spec ends with a **docs write-back story** after runtime validation that brings every affected doc to shipped reality in that doc set's conventions, and graduates cross-sprint `lessons.md` entries into the `rule_docs`. Name the seed targets in its notes. No doc left wrong → no story.

## Phase 4: External pressure-test

After writing the spec, suggest `multi-llm-review` in plan mode in one line (two external models challenge the spec in parallel) and leave the call to the user. Never run it yourself. If run: accepted → spec edit via `spec-json` jq; deferred → `interviewSummary.risks`; rejected → rationale in `interviewSummary.decisions`. It hardens the plan; it doesn't gate the loop.

## Regenerate mode

Keep `passes: true` stories untouched. Re-evaluate incomplete stories against the current PRD and `lessons.md`, keeping IDs stable. Add or remove stories per scope changes and re-validate every `deps` chain. Suggest Phase 4 again only if the story set materially changed.

## lessons.md

Create `<sprint_root>/{sprint}/lessons.md` if absent:

```markdown
# Lessons - {sprint-name}

> High-signal knowledge for future work in this sprint. Story-local context belongs in `spec.json` `notes`, not here.
> Only add entries that would help future work avoid a problem or rediscovery — durable, non-obvious, and not already in `notes` / code / architecture decisions / docs.

## Planning Context

- {Codebase patterns discovered, runtime constraints, key interview decisions that affect implementation}

---

## Gotchas & Patterns

_Append new entries at the END. Heading: `### <short topic> — <STORY_ID>`. Never edit or reorder existing entries._
```

## Before finishing

- Every design-shaping decision was ratified by the user; none first appears in the spec.
- Every story ID cited in a description or note resolves to the story that owns that behaviour.
- `interviewSummary` holds the scope changes, risks and decisions; `branch` is `sprint/{NNN}-{slug}`.

## Output

1. Write `spec.json` and `lessons.md`.
2. Show:

```
Sprint: {NNN}-{slug}
Branch: sprint/{NNN}-{slug}
Stories: {count} ({testFirst count} test-first, {review count} checkpoints)

| ID | Title | Deps | testFirst | review |
|----|-------|------|-----------|--------|
```

3. Confirm the spec with the user and suggest the Phase 4 pressure-test.

<promise>PLAN_DONE</promise>
