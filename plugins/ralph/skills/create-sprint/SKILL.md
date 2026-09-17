---
name: create-sprint
description: "Carve a new sprint into {sprint_root}/{NNN}-{slug}/PRD.md: read the docs and code as the baseline, design past them with the user, and write the plan of record ralph-plan decomposes. Use when defining a new implementation sprint."
---

# Create Sprint — Sprint PRD

Creates `<sprint_root>/{NNN}-{slug}/PRD.md` — the plan of record a future Ralph loop plans from. This skill does **not** create `spec.json` or `lessons.md`; `ralph-plan` owns those.

You are carving scope, not designing implementation. Short and sharp.

## Repo config

Read `<repo>/.agents/config.yaml` first (see `~/.claude/ralph/README.md` for the full contract). Keys used here:

| Key | Use | Default if absent |
| --- | --- | --- |
| `sprint_root` | where sprint folders live | `sprints` |
| `docs_root` | the product + technical baseline docs | none — fall back to the repo's `README`/`CLAUDE.md` and say so |
| `glossary` | canonical vocabulary to inherit | none |
| `adr_log` | settled architecture decisions | none |
| `rule_docs` | the repo's own rule surfaces | `CLAUDE.md` |

No config file → state the defaults you assumed in your first message, then continue.

## Docs are the baseline, not the ceiling

`docs_root` and the prior sprints record what the product was meant to be and what actually got built. Read them to inherit vocabulary, settled decisions, and the shape things were supposed to fit — then design past them. They were written before the last few sprints taught what they taught; a sprint that only transcribes a documented slice leaves the best ideas on the table. Bring what the docs did not think of to the interview as proposals, recommendation attached, and let the user decide what's in. Once written, the PRD is the plan of record: a reference or roadmap doc that disagrees with it is stale, and the sprint's closing write-back fixes the doc. The one exception is a settled architecture decision (`adr_log`) or a safety invariant — a sprint changes one only through `ralph-plan`'s ADR gate, never by a PRD line alone.

## Input & Resolution

```
/create-sprint 040-observability-and-ops
/create-sprint observability and ops
/create-sprint
```

1. Inspect `<sprint_root>/` for existing numbering.
2. Exact `NNN-slug` given → validate uniqueness and use it.
3. Theme only or nothing → propose the next zero-padded number + a slug from the theme; ask a short follow-up only if genuinely ambiguous.

## Context Gathering

Before any question:

1. **Explore the code the sprint touches** — what already exists, what's half-built, what the theme assumes is there. A PRD that scopes already-built work or assumes a missing foundation sends the whole loop into premise-failure BLOCKED signals; sprint creation is the cheapest place to catch that. Delegate broad sweeps to Explore subagents.
2. **Read the docs for the area** — the relevant files under `docs_root`, the `glossary`, the `adr_log`, and the roadmap doc where the area has an entry. Note where a doc's status or content is already behind the code; those notes seed the planner's write-back decision.
3. **Mine the sprints that built the area** — `spec.json` handoffs and `lessons.md` (`spec-json` skill) for constraints and landmines worth carrying into Ralph Notes, `FOLLOWUP.md` for deferred items that may belong here, adjacent PRDs' Risks and open-gap statements for seams this sprint could close. Check the repo's one-off plans folder and open branches for parallel work on the same surface.
4. **Form the sprint's own ideas.** With code and docs in view, ask what would make this slice genuinely good rather than merely documented: what became cheap since the docs were written, what a first-time user actually needs, what an adjacent sprint left unowned.

Targeted, not exhaustive — you're carving a sprint, not writing a product spec.

## Interview

Open with your read of the sprint — why now, the proposed scope boundary, the single most important outcome, and the ideas beyond the docs you'd add — then carve the scope with the user, not for them: every scope-shaping call (what's in, what's deferred, what proof the sprint owes) is theirs to ratify, surfaced as it arises with your recommendation attached. The written PRD must contain no boundary the user hasn't seen.

Ground the PRD must cover before it's written — resolved by exploration where possible, by question where not:

- **Placement**: why now, what comes immediately before and after
- **Scope boundary**: in vs explicitly deferred; the single most important outcome
- **Concrete outcomes** per touched package or surface
- **External dependencies**: services to deploy, credentials, third-party APIs, infra changes
- **Live validation**: which environments must prove the sprint, what evidence counts as complete
- **Assumptions** needing early validation (spike candidates), and whether the sprint changes an earlier sprint's assumptions or inherits its deferred work

How to interview:

- **One question at a time, with your recommended answer attached.** The user confirms, rejects, or refines — never a blank prompt. Batch only questions that are genuinely independent.
- **Explore instead of asking** whenever the codebase, docs, or an earlier sprint can answer.
- **Use the canonical vocabulary** — the terms in `glossary`. A concept the sprint introduces gets a name and a one-line definition in the PRD; it reaches the glossary at write-back.
- **Pin the deliverables to concrete outputs.** Not "implement package X" — "the schema for the snapshot table", "calldata builders unit tested against known-good fixtures".
- **Force one concrete failure scenario for the validation plan** ("the deploy dies halfway — what's the fallback?"). A vague answer means the runtime validation plan isn't ready to write down.
- **Cross-check against `adr_log` and the safety invariants in `rule_docs`.** A sprint that wants to change one says so in Risks or Ralph Notes so `ralph-plan`'s ADR gate records the decision — never smuggled in as an outcome.
- **Stop when scope, non-goals, validation, and exit criteria are resolved** — or when answers turn speculative. Speculation becomes an Integration Spike row or a Risks entry, not more questions.

## Sprint PRD Contract

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

- **Thin.** Point at what the docs already say; write down what's new — the sprint's own decisions, corrections to the docs' picture of the code, the ideas the docs lacked. The PRD tells Ralph what the sprint is trying to prove, not re-teach the system.
- **Outcomes, not designs.** In-Scope items name capabilities, surfaces, and proof — concrete enough that `ralph-plan` decomposes without interpreting abstract language, but never internal design (file layouts, function names, data shapes, chosen algorithms). Design belongs to `ralph-plan`'s interview and the builder; an implementation choice that genuinely must be fixed this early is an architecture decision to surface explicitly, not a PRD line.
- **Greenfield default where the repo is pre-launch.** No external users: scope outcomes to the cleanest end-state — reshape data models, rework components, delete superseded surfaces as if the sprint's design had been there from the start. "Keep X working unchanged" appears in a PRD only as an explicit user decision (a kill-switch semantic, an external contract), never as an assumed constraint the PRD invents. A repo with live users inverts this default; the user says which applies.
- **Sized for one Ralph loop run.** If the outcomes won't decompose into one coherent story sequence, propose splitting into two sprints instead of writing a mega-PRD.
- At least one real runtime validation requirement — even foundational sprints get an explicit live proof (smoke deploy, staging deploy, branch migration, live auth check).
- Exit criteria each falsifiable, naming the evidence that proves it, plus the sprint-wide ones (CI green, deployed and verified).
- **Ralph Notes** carry loop-facing context that fits nowhere else: landmines mined from prior sprints' lessons/handoffs, constraints the user insists on, inherited `FOLLOWUP.md` items now in scope, pointers the planner must not miss.

## Output

1. Create the folder, write `<sprint_root>/{NNN}-{slug}/PRD.md`.
2. Summarize: sprint name, file path, the outcomes that go beyond the docs, live validation included.
3. Suggest `ralph-plan` next when the user wants `spec.json`.
