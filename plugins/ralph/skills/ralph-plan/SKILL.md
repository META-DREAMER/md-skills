---
name: ralph-plan
description: "Generate spec.json from a sprint PRD: research, a pair-design conversation on the real forks, then a plan of record. Use when preparing a sprint for autonomous execution."
---

# Ralph Plan — Spec Generation

Turns `<sprint_root>/{sprint}/PRD.md` into an executable `spec.json`. You are planning for a builder as capable as you: the spec is a contract of **intent and evidence**, not an instruction manual. Prescribe WHAT each story delivers and how it's proven; leave HOW to the builder.

## Repo config

Read `<repo>/.agents/config.yaml` first (contract: `~/.claude/ralph/README.md`). Keys used here: `sprint_root` (default `sprints`), `docs_root`, `glossary`, `adr_log`, `rule_docs` (default: the repo's `CLAUDE.md`). No config file → state the defaults you assumed, then continue.

## Source of Truth

The PRD is the plan of record for this sprint. `docs_root` is the baseline it extends: read it for vocabulary, settled decisions, and how things were meant to fit, then plan the sprint as written. A reference or roadmap doc that disagrees with the PRD is stale — note it for the write-back story, don't plan around it. Settled decisions in `adr_log` and the safety invariants in `rule_docs` are the exception: changing one goes through the ADR gate below with the user's ratification, never silently. If the code contradicts the PRD's premise, surface that to the user.

## Input & Resolution

```
/ralph-plan 020
/ralph-plan 020-execution-backbone
```

Prefix → match `<sprint_root>/020-*`; no match → error and list available sprints.

**Mode:** `spec.json` already exists → **Regenerate** (see below). Only `PRD.md` → **Normal**.

## Phase 1 — Research

Read before forming any opinion:

1. `<sprint_root>/{sprint}/PRD.md` and every doc it lists under Context.
2. `glossary` and `adr_log` — hold these as the canonical vocabulary and constraints for everything that follows.
3. `<sprint_root>/{sprint}/lessons.md` if it exists, and — for the areas this sprint touches — earlier sprints' `spec.json` handoffs (`spec-json` skill), `lessons.md`, and `FOLLOWUP.md` (deferred items there may belong in this sprint's stories).
4. **Cross-sprint synthesis:** scan adjacent sprints' PRDs — especially their Risks and open-gap statements — for seams this sprint's deliverables are uniquely positioned to close (e.g. another sprint flagged "no atomic producer seam for X" and this sprint is building exactly that state machine). A closable gap becomes an explicit story or a named deferral in `interviewSummary` — never silence. Note execution order too: a planned-but-unbuilt sprint that lands **before** this one executes is part of this sprint's baseline — read its PRD/spec and plan against the codebase as it will be, keeping integration with its deliverables deliberately high-level (its shape may still drift).
5. The code the sprint touches. Verify the PRD's claims about "how it works today" against the actual implementation — delegate broad exploration to subagents, quote file/line when the code disagrees with the PRD or the docs.
6. Third-party surfaces: verify library/protocol signatures against reality — the installed package's types/source, official docs online, or Context7 MCP when available. Training data is stale for version-sensitive APIs.

Come out of this with: what the sprint delivers, the two or three real design forks, and what the PRD leaves undecided.

## Phase 2 — Design conversation (pair-design, recommend-first)

Planning is a pair-design conversation, not a report. Open with your understanding — deliverables, candidate approaches, the tradeoffs between them — then design out loud: narrate how the sprint is being shaped and wired as you work it out, and put each fork to the user as it arises. The first time the user sees a decision must never be the finished spec.json.

**Any decision that shapes the sprint's architecture, data model, integration posture, scope, or proof bar is the user's to ratify.** You attach a recommendation and the why; the user confirms, rejects, or refines. Confidence is not a license to skip the checkpoint — the decisions that go wrong are the ones a planner finds obvious: whether an integration warrants its own package, who advances long-running state (client polling vs a server-side scheduler), extend a shape vs reshape to the end-state (pre-launch: reshape is the default to recommend), wire a cross-sprint seam now vs a named deferral, how much live proof — and spend — the sprint buys. Illustrations, not a checklist: the class is any choice where a different answer yields a different sprint.

- **Every question carries your recommended answer.** The user reacts to a position, never a blank prompt.
- Don't ask what docs, code, or the PRD already answer — explore instead. If the sprint-PRD interview happened earlier in this session, its answers stand; don't re-ask.
- Batch only genuinely parallel questions — independent forks that don't reshape each other. A decision whose answer changes the next question is asked alone, when it arises.
- Challenge terms against the `glossary`; cross-reference claims against code and `adr_log`; probe with a concrete failure scenario where the answer would change the plan ("the job fires twice for the same record — whose story handles that?").
- A question that only yields speculation isn't a question — it's an **integration spike story**. Route it there and move on.

## Phase 3 — Plan of record, then grill the residue

Draft the full story breakdown from the ratified decisions. Apply the **deletion test** to each story: if deleting its deliverable makes no complexity reappear anywhere, it's a pass-through — fold it into a neighbour. Favor stories that produce deep modules (high leverage behind a small interface); the repo's `rule_docs` carry its own version of this principle — use it. The deletion test scopes *stories and helpers* — package boundaries were ratified with the user in Phase 2; don't re-litigate them here.

Present the plan of record (story list with deps, checkpoint placement, risks). Then resolve any **remaining genuine forks one question at a time**, recommended answer first. Stop when the open branches are resolved or the only questions left are speculative — those become spikes or `interviewSummary.risks`, not more talking.

**ADR gate (before writing spec.json):** if any interview decision is all three of — hard to reverse, surprising without context, the result of a real trade-off — record it in `adr_log` in that file's existing format (next number, the decision, why it was chosen, who owns the detail), plus a short subsection in the owning spec doc if the why needs more than a line. No `adr_log` configured → record the decision in `interviewSummary.decisions` and tell the user the repo has nowhere durable to put it. Otherwise skip; don't manufacture ADRs.

**Inline doc updates (not batched):** a term sharpened during the interview that outlives the sprint → propose the `glossary` patch in that turn. Sprint-local discoveries → `lessons.md` Planning Context.

## Spec Generation

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

Story IDs are `{NNN}-{XX}` (sprint number zero-padded to 3, sequence to 2): `076-01`, `076-15`.

### Intent, not instructions

`description` carries what and why — scope boundaries, the contract with neighbouring stories. `notes` carries only what the builder can't derive: non-obvious gotchas, specific doc sections, known edge cases. **Never step-by-step implementation** — the builder reads the `rule_docs`, the owning package's own rules, and neighbouring code itself; a spec that dictates HOW handcuffs it into a worse solution than it would find alone. One sanctioned exception: `difficulty: "low"` stories get prescriptive notes (see the difficulty section).

Calibrate specificity by what goes stale: cross-story contracts and interview-settled decisions are always pinned in criteria/notes, with the why; internals stay intent-only unless the story is `low`. **Late-sprint stories lean intent, not internals** — drift risk grows with distance, so plan-time specifics are stalest exactly where stories run last.

### Sizing — judgment, not line counts

A story is **one coherent unit of intent, independently provable, worth one loop iteration's overhead** (pick, build, prove, handoff, polish, commit). Err toward fewer, deeper stories — the builder delegates to subagents and handles large diffs; over-fragmentation costs a full iteration per fragment and scatters what should be one design decision. Split when a story mixes concerns that are proven differently (schema vs UI vs live validation), or when a protected-surface change is bundled with unrelated work.

### Ordering — dependency-driven, vertical first

Two hard invariants: **integration spikes first** (runtime-compatibility questions that could change the plan), **runtime validation last**. Between them, order by real dependency and prefer an early **vertical slice** — one thin end-to-end path through the sprint's architecture — so integration surprises surface in iteration 3, not iteration 20. Broaden layer-by-layer only after the skeleton stands.

`deps` rules: a story may only depend on lower sequence numbers; no cycles; `[]` when independent (the loop can parallelize those).

### Acceptance criteria — every criterion names its evidence

Typecheck, lint, and format are enforced by the pre-commit hook and PR CI, and every story runs its own tests — **don't restate them as criteria**. Each criterion states what is proven and by what: a test name or behavior, a command plus expected output, a `browser-verify` check, a read from the live environment. Criteria naming a live surface are proven ON that surface — deterministic green doesn't satisfy them.

Bad: "Works correctly." Good: "Returns health factor 1.5 for a position with $1500 collateral and $1000 debt."

Typical evidence by story kind (reference, not a quota): pure logic / state machines → named tests covering the transitions; API endpoints → request/response validation tests; UI → `browser-verify` check; deployment → deployed target + captured response; integration spike → the question answered with the outcome recorded, fallback documented if it failed.

### testFirst

`true` for pure logic (calculations, encoding, domain math), state machines, crypto/signing round-trips, API endpoints, bug fixes (regression test first). `false` for schema/migrations, UI components, config/infra, wiring, deployment.

### difficulty

Sets how much reasoning effort the implementing agent spends on the story. Absent = `med`.

- `low` — the shape is obvious: wiring, config, mechanical UI. Executed at reduced effort, so the planner de-risks by writing **more prescriptive notes** — concrete files, functions, expected shapes. This is the sanctioned exception to intent-not-instructions; the why still comes along.
- `med` (default) — standard effort, intent-only notes.
- `high` — judgment-heavy or novel: state reconcilers, protocol encoding, cross-service protocols, tricky migrations. High effort; the builder implements first, then gets one round of adversarial review on the actual diff (see the build prompt).

Protected-surface stories — whatever the repo's `CLAUDE.md` marks as protected — are never `low`.

### review Flag (Review Checkpoints)

`review: true` marks a **review checkpoint**: after that story completes, the loop runs the `story-review` skill over the *cumulative diff since the last checkpoint* — not just that story. Review is expensive and noisy when run per-story; checkpoints batch it so each review sees cross-story interactions.

Chunk reviews by judgment, not a quota — place a checkpoint wherever the accumulated diff forms a coherent, reviewable unit. A sprint of small wiring stories might need only a couple; a sprint of large risky stories might warrant one per story. Natural checkpoint locations:

- After a cluster of risky stories completes (execution paths, state machines, schema changes with consumers)
- At layer/phase boundaries (e.g. last backend story before UI begins)
- On any single story big or risky enough to deserve its own review
- **Always** on the final implementation story before runtime validation — nothing reaches the PR unreviewed

Don't agonize over placement: the loop also self-triggers a checkpoint whenever a story's diff touches a protected surface, so the flag is a cadence device, not the safety net.

### Runtime-proven sprint rule

If the PRD requires real-environment proof, the spec MUST carry it as explicit first-class stories (staging deploy + verification, live migration, live auth check) — never buried as a vague criterion on an unrelated story. The final validation story writes `<sprint_root>/{sprint}/VALIDATION.md` — never repo-root files. Repos with their own live-validation skill route that story through it.

### Docs write-back closing rule

Sprints run ahead of `docs_root` by design, so decide at plan time whether this one leaves the docs wrong: a doc describing a surface the sprint reshapes, a status marker it flips, an architecture decision the interview changed, or a doc the PRD's Context already flags as stale. If so, the spec MUST end with a **docs write-back story** after runtime validation (so it records what shipped, not what was planned): update every affected doc to shipped reality — terse, following that doc set's own conventions — plus any drift discovered during the sprint, and graduate cross-sprint `lessons.md` entries into the `rule_docs`. Name the seed targets in the story's notes. A sprint that leaves no doc wrong skips it; don't manufacture the story.

## Phase 4 — External pressure-test (suggested, never auto-run)

After the draft `spec.json` is written, suggest a `multi-llm-review` of the plan to the user — one line on what it buys (two external models challenge the spec in parallel; independent convergence on a defect is the highest-signal output) — and leave the call to them. Don't invoke it yourself: it spends external-model time and ends in a disposition round, both the user's to opt into. If they run it, dispositions land per that skill: accepted → concrete spec edit (`spec-json` jq); deferred → `interviewSummary.risks`; rejected → rationale in `interviewSummary.decisions` so the loop never re-argues it. Declined or no challenger available → the spec stands as-is; the pressure-test hardens the plan, it doesn't gate the loop.

## Regenerate Mode

When `spec.json` exists: preserve `passes: true` stories untouched; re-evaluate incomplete stories against the current PRD and `lessons.md` (keep IDs stable); add/remove stories per interview scope changes; re-validate every `deps` chain (no dangling references). Suggest the Phase 4 pressure-test again only when the story set materially changed.

## Lessons File

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

## Output Checklist

- [ ] Story IDs unique, `{NNN}-{XX}`; deps reference valid earlier IDs, no cycles
- [ ] Spikes first, runtime validation last, early vertical slice where the sprint allows one
- [ ] Every criterion names its evidence; live-surface criteria exist wherever the PRD demands live proof
- [ ] Descriptions carry intent/boundaries — no step-by-step implementation anywhere
- [ ] `testFirst` and `review: true` checkpoints set per the rules above (final implementation story always a checkpoint)
- [ ] `difficulty` set per story nature; protected-surface stories never `low`; `low` stories carry prescriptive notes
- [ ] Every design-shaping decision put to the user with a recommendation and ratified — none decided silently, none first seen in the spec
- [ ] ADR gate applied; inline doc updates proposed where terms sharpened
- [ ] Docs write-back closing story present when the sprint leaves any `docs_root` doc wrong (surface reshaped, status flipped, decision changed)
- [ ] Cross-sprint gaps this sprint can close are stories or named deferrals; every story ID cited in a description/note resolves to the story that owns that behavior
- [ ] `multi-llm-review` of the plan suggested to the user (their call to run); if run, dispositions applied and recorded in `interviewSummary`
- [ ] `interviewSummary` captures scope changes, risks, and the decisions that shaped the plan
- [ ] `lessons.md` created/updated; `branch` follows `sprint/{NNN}-{slug}`

## Final Output

1. Write `<sprint_root>/{sprint}/spec.json` and `<sprint_root>/{sprint}/lessons.md`.
2. Display a summary table:

```
Sprint: {NNN}-{slug}
Branch: sprint/{NNN}-{slug}
Stories: {count} ({testFirst count} test-first, {review count} checkpoints)

| ID | Title | Deps | testFirst | review |
|----|-------|------|-----------|--------|
```

3. Confirm with the user that the spec looks correct, and suggest the optional external pressure-test before the loop runs (Phase 4).

<promise>PLAN_DONE</promise>
