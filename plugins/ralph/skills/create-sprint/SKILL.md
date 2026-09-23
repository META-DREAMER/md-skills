---
name: create-sprint
description: "Carve a new sprint into {sprint_root}/{NNN}-{slug}/PRD.md: read the docs and code as the baseline, design past them with the user, and write the plan of record ralph-plan decomposes. Use when defining a new implementation sprint."
---

# Create Sprint

Writes `<sprint_root>/{NNN}-{slug}/PRD.md`, the plan of record a Ralph loop plans from. It does not create `spec.json` or `lessons.md`; `ralph-plan` owns those. You carve scope, not implementation.

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

## Docs are the baseline

Read `docs_root` and prior sprints for vocabulary, settled decisions and intended shape, then design past them: they predate what recent sprints learned. Bring ideas the docs lack to the interview as proposals with a recommendation. Once written, the PRD is the plan of record; a reference or roadmap doc that disagrees is stale and the sprint's write-back fixes it. Exception: a settled decision in `adr_log` or a safety invariant changes only through `ralph-plan`'s ADR gate, never by a PRD line.

## Input

```
/create-sprint 040-observability-and-ops
/create-sprint observability and ops
/create-sprint
```

1. Inspect `<sprint_root>/` for existing numbering.
2. Exact `NNN-slug` → check it is unique and use it.
3. Theme or nothing → propose the next zero-padded number and a slug; ask only if genuinely ambiguous.

## Context gathering

Before any question:

1. **Explore the code the sprint touches**: what exists, what is half-built, what the theme assumes. A PRD that scopes built work or assumes a missing foundation sends the loop into premise-failure BLOCKED signals. Delegate broad sweeps to Explore subagents.
2. **Read the docs for the area**: the relevant files under `docs_root`, `glossary`, `adr_log`, and the `roadmap` entry if any. Note where a doc is already behind the code; that seeds the write-back decision.
3. **Mine the sprints that built the area**: `spec.json` handoffs and `lessons.md` (`spec-json` skill) for landmines worth carrying into Ralph Notes, `FOLLOWUP.md` for deferred items, adjacent PRDs' Risks for seams this sprint could close. Check the repo's one-off plans folder and open branches for parallel work on the same surface.
4. **Form the sprint's own ideas**: what became cheap since the docs were written, what a first-time user needs, what an adjacent sprint left unowned.

Targeted, not exhaustive.

## Interview

Open with your read: why now, the proposed boundary, the single most important outcome, and the ideas beyond the docs. Every scope-shaping call (what's in, what's deferred, what proof the sprint owes) is the user's to ratify; the PRD contains no boundary the user hasn't seen.

The PRD must cover, by exploration where possible and by question where not:

- **Placement**: why now, what comes before and after
- **Scope boundary**: in vs deferred; the single most important outcome
- **Concrete outcomes** per touched package or surface
- **External dependencies**: services to deploy, credentials, third-party APIs, infra changes
- **Live validation**: which environments prove the sprint and what evidence counts
- **Assumptions** needing early validation (spike candidates); earlier sprints' assumptions changed or deferred work inherited

How:

- **One question at a time, with your recommended answer.** Batch only genuinely independent questions.
- **Explore instead of asking** whenever code, docs or an earlier sprint can answer.
- **Use the `glossary` terms.** A new concept gets a name and a one-line definition in the PRD; it reaches the glossary at write-back.
- **Pin deliverables to concrete outputs**: "the schema for the snapshot table", not "implement package X".
- **Force one concrete failure scenario for the validation plan** ("the deploy dies halfway: what's the fallback?"). A vague answer means the plan isn't ready.
- **Cross-check against `adr_log` and the safety invariants in `rule_docs`.** A sprint that changes one says so in Risks or Ralph Notes so the ADR gate records it.
- **Stop when scope, non-goals, validation and exit criteria are resolved**, or answers turn speculative. Speculation becomes an Integration Spike row or a Risk.

## PRD contract

Create only `<sprint_root>/{sprint}/PRD.md`:

```markdown
# Sprint PRD: {NNN}-{slug}

## Purpose / Why Now
[Short paragraph: the gap, why this sprint, what it unlocks.]

## Context
[The reading list — docs, prior sprints, plans, code — one line each on what it contributes. Flag docs already behind the code.]

## In-Scope Outcomes
- ...

## Explicit Non-Goals
- ...

## Dependencies on Earlier Sprints
- ...

## Runtime Validation Plan
- ...

## Integration Spikes
{Only when the sprint touches new runtimes or unproven dependencies. Omit if none.}
| Spike | Question It Answers | Pass Criteria | Fallback If Fails |
|-------|---------------------|---------------|--------------------|

## Sprint Exit Criteria
- ...

## Risks / Assumptions To Validate
- ...

## Ralph Notes
- ...
```

Rules:

- **Thin.** Point at what the docs say; write down only what's new: the sprint's decisions, corrections to the docs' picture of the code, ideas the docs lacked.
- **Outcomes, not designs.** In-Scope items name capabilities, surfaces and proof, concretely enough that `ralph-plan` decomposes them without interpretation, but never file layouts, function names, data shapes or algorithms. A choice that must be fixed this early is an architecture decision to surface explicitly.
- **Greenfield default for a pre-launch repo.** Scope to the cleanest end-state: reshape data models, delete superseded surfaces. "Keep X working unchanged" appears only as an explicit user decision. A repo with live users inverts this; the user says which applies.
- **Sized for one loop run.** If the outcomes won't form one coherent story sequence, propose two sprints.
- At least one real runtime validation requirement, even for foundational sprints (smoke deploy, staging deploy, branch migration, live auth check).
- Exit criteria are falsifiable and name their evidence, plus the sprint-wide ones (CI green, deployed and verified).
- **Ralph Notes** carry loop-facing context that fits nowhere else: landmines from prior lessons and handoffs, user constraints, inherited `FOLLOWUP.md` items, pointers the planner must not miss.

## Output

1. Write `<sprint_root>/{NNN}-{slug}/PRD.md`.
2. Summarize: sprint name, path, outcomes beyond the docs, live validation included.
3. Suggest `ralph-plan` next.
