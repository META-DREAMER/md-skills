---
name: multi-llm-review
description: "Pressure-test a plan document or a branch diff with external models — Codex (openai-codex plugin) and Gemini (gemini-cli MCP) in parallel — then verify every finding yourself, synthesize, and disposition with the user. Use when asked to pressure-test, red-team, or get outside-model review of a design, plan, or branch."
---

# Multi-LLM Review

Two external challengers review the same artifact in parallel: **Codex** (deep verification — it reads the repo itself and checks claims against reality) and **Gemini** (taste — over-engineering, gaps, simpler designs; fed the full dossier because it cannot research). Their prompts share one core rubric on purpose: **where both models independently flag the same defect is the highest-signal output of this skill**.

You are the synthesizer, not a relay. Every finding must survive your own verification before the user sees it, and the user decides every disposition. A clean run is a successful run.

## Configuration

Read `<repo>/.agents/config.yaml` when it exists. Used here:

| Key | Use |
|-----|-----|
| `glossary`, `adr_log` | Dossier files, and the settled-decisions list the prompts must not relitigate |
| `docs_root` | Where the project's spec docs live, for the doc-contradiction rubric item |
| `sprint_root`, `worklog_root` | Where the plan document and the review-state file live |

Absent config: the plan document is whatever the user names, dossier is the docs they point at, review state goes next to it, and say in the report that you used defaults. A project may ship its own `multi-llm-review` overlay naming its plan format and state files; load it too when present.

## Modes

- **plan** — reviews a plan document (a spec, PRD, design doc, or structured story breakdown) *before* it executes. Default when the branch has no commits beyond its base.
- **code** — reviews the cumulative branch diff against the plan's acceptance criteria. Default when the branch has commits and the work is (mostly) complete. Ambiguous → ask.

## 1. Detect challengers

Per `references/invocation.md`: Codex via `CLAUDE_PLUGIN_ROOT` or the plugin cache glob; Gemini via ToolSearch + `ping`. Print detection status. One missing → warn (`{model} skipped: {reason}`) and proceed single-model. Both missing → stop and report. There is no point running this skill as a self-review.

## 2. Assemble the dossier

What the models judge against — nothing more, nothing less.

- **plan**: the plan document and the requirements doc behind it, the glossary, the architecture-decision log, any lessons file's planning context, and any doc the plan lists as context.
- **code**: `git diff <base>...HEAD` written to a file, the plan's acceptance criteria, its exit criteria, and the open-follow-ups file if one exists.

## 3. Fan out — both models, in parallel, one message

Prompts in `references/prompts.md`; call mechanics in `references/invocation.md`. Codex runs read-only (**never `--write`**), model and effort unset. Gemini runs with `changeMode: false`, model unset. Launch the Codex job and the Gemini call together. Ceiling: 20 minutes in plan mode, 10 in code mode — at the ceiling ask the user whether to keep waiting. Verify the other model's findings while Codex runs.

**Challenger outages are normal.** Brief each challenger to write its result to a file in the scratchpad, and read the file. A model that dies mid-answer, returns markdown where JSON was demanded, or leaks backend noise is an expected outcome, not a reason to abandon the run.

## 4. Synthesize and verify

1. Parse both outputs into the shared shape (`verdict`, `findings[]` with severity / target / confidence).
2. Dedupe: same target + same defect → one finding tagged `[codex+gemini]`. Convergence raises priority, not truth — it still gets verified.
3. **Verify every finding yourself.** The burden of proof is on the challenger: what would actually break, today, if this shipped as-is? Actively try to refute it — read the cited code, check whether a pending item already owns the gap, check whether a recorded decision already settled the question.
4. Classify: **CONFIRMED** (a concrete failing scenario you can trace) / **PLAUSIBLE** (unrefuted **and** touching an invariant — money, idempotency, auth, data loss, credentials; otherwise drop) / dropped (report as one count, not item by item).

Never manufacture findings to justify the run. Zero survivors → report `REVIEW_CLEAN` with the scope and move on.

## 5. Disposition round — the user decides

One batched round, recommendation first. Findings are independent forks, so they batch well. Give each surviving finding with its source(s), severity, your verdict, and a **recommended disposition**:

- **accept** — state the concrete change it implies; don't make the user derive it.
- **defer** — into the project's risks or follow-ups file.
- **reject** — with the rationale recorded, so nothing re-argues it later.

A finding that only yields speculation under discussion is not a disposition question. Route it to a spike or a risk entry and move on. Don't relitigate what the user already decided this session.

## 6. Apply

- **plan**: accepted findings edit the plan document. Deferred ones go to its risks section. Rejected ones get a one-line decision entry naming why, so a later pass doesn't reopen them.
- **code**: **never fix code from this skill.** Review output is not a license to edit. Accepted findings become new work items or follow-up entries; the user picks which.

## 7. Report

```
MULTI-LLM REVIEW — {target} ({mode})
Challengers: codex ✓ gemini ✓        Dropped in verification: {n}

| # | Source(s) | Sev | Target | Finding | Verdict | Disposition |
|---|-----------|-----|--------|---------|---------|-------------|
```

Add one calibration line. Both models returning `approve` with zero findings *every* run means the prompts have gone soft; healthy challengers find something material a meaningful fraction of the time. Zero survivors on *this* run is success.

## Never

- Run Codex with `--write`, or edit any file as a side effect of review. Plan edits happen only after the user dispositions.
- Present an unverified model finding as fact. Models hallucinate file paths, identifiers, and constraints.
- Reopen a settled decision on a model's preference alone. A concrete named failure is the only key that reopens one.
- Substitute this skill for the project's own code review or quality gate. It complements them at plan time and pre-PR.
