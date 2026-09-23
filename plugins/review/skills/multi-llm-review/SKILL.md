---
name: multi-llm-review
description: "Pressure-test a plan document or a branch diff with external models, Codex (openai-codex plugin) and Gemini (gemini-cli MCP) in parallel, then verify every finding yourself, synthesize, and disposition with the user. Always user-invoked. Use when asked to pressure-test, red-team, or get outside-model review of a design, plan, sprint spec or branch."
---

# Multi-LLM Review

Two external challengers review the same artifact in parallel. **Codex** verifies: it reads the repo and checks claims against reality. **Gemini** judges taste (over-engineering, gaps, simpler designs) from a full dossier, because it cannot research. Their prompts share one rubric on purpose: **a defect both models flag independently is the highest-signal output.**

You are the synthesizer, not a relay. Every finding survives your own verification before the user sees it, and the user decides every disposition. A clean run is a successful run.

## Repo facts

**Repo facts.** Resolve each fact below in order: its `.agents/config.yaml` key; what the project's `CLAUDE.md`, `AGENTS.md` or README names; the conventional places listed. Relative paths resolve from the repo root. A fact you only read and cannot find is skipped with a note; one you must write is created at the default shown. Never guess a command. Say what you resolved, and from where, in your first status line.

| Fact | Key | Look for | If none |
| --- | --- | --- | --- |
| Product and technical docs | `docs_root` | the docs `CLAUDE.md` or README points to; `docs/` | work from the code and note the gap |
| Glossary | `glossary` | a glossary or terminology section the project names; `docs/**/glossary*.md` | read: skip. Write-back: create `docs/glossary.md` |
| Architecture decisions | `adr_log` | `docs/adr/`, `docs/decisions*`, `DECISIONS.md`, `ADR*.md`, an architecture-decisions doc | read: skip. Recording one: create `docs/decisions.md` |
| Sprint folders | `sprint_root` | an existing `sprints/` | create `sprints/` |
| Worklogs | `worklog_root` | an existing `worklogs/` | create `worklogs/` |

## Modes

- **plan** reviews a plan document (spec, PRD, design doc, story breakdown) before it executes. Default when the branch has no commits beyond its base.
- **code** reviews the cumulative branch diff against the plan's acceptance criteria, before the PR. Default when the branch has commits and the work is mostly complete. Ambiguous: ask.

**Sprint targets.** `/multi-llm-review <sprint> [plan|code]` resolves the sprint by prefix match under `sprint_root`. The plan is its `spec.json` and `PRD.md`; `interviewSummary.decisions` are settled decisions; deferrals go to `interviewSummary.risks` (plan) or `FOLLOWUP.md` (code); accepted plan edits go through `spec-json`'s jq rule, never a whole-file write. `ralph-plan` suggests plan mode and `ralph-loop` suggests code mode; neither runs it.

## 1. Detect challengers

Per [invocation.md](references/invocation.md): Codex via `CLAUDE_PLUGIN_ROOT` or the plugin cache glob; Gemini via ToolSearch + `ping`. Print detection status. One missing: warn (`{model} skipped: {reason}`) and proceed single-model. Both missing: stop and report. This skill is never a self-review.

## 2. Assemble the dossier

What the models judge against, nothing more:

- **plan**: the plan and the requirements doc behind it, the glossary, the decision log, any lessons file's planning context, and any doc the plan lists as context.
- **code**: `git diff <base>...HEAD` written to a file, the plan's acceptance and exit criteria, and the open-follow-ups file if one exists.

## 3. Fan out, both models in one message

Prompts: [prompts.md](references/prompts.md). Codex runs read-only (never `--write`), model and effort unset. Gemini runs with `changeMode: false`, model unset. Ceiling: 20 minutes in plan mode, 10 in code mode; at the ceiling, ask the user whether to keep waiting. Verify one model's findings while the other runs.

**Challenger outages are normal.** Have each challenger write its result to a scratchpad file and read the file. A model that dies mid-answer, returns markdown instead of JSON, or leaks backend noise is an expected outcome, not a reason to abandon the run.

## 4. Synthesize and verify

1. Parse both outputs into the shared shape (`verdict`, `findings[]` with severity, target, confidence).
2. Dedupe: same target and same defect becomes one finding tagged `[codex+gemini]`. Convergence raises priority, not truth.
3. **Verify every finding yourself.** The burden of proof is on the challenger: what would break today if this shipped as-is? Try to refute it: read the cited code, check whether a pending item owns the gap, check whether a recorded decision settled it.
4. Classify: **CONFIRMED** (a failing scenario you can trace); **PLAUSIBLE** (unrefuted and touching an invariant: money, idempotency, auth, data loss, credentials); otherwise dropped, reported as one count.

Never manufacture findings. Zero survivors: report `REVIEW_CLEAN` with the scope.

## 5. Disposition round

One batched round, recommendation first. Each surviving finding with source(s), severity, your verdict and a recommended disposition:

- **accept**: state the concrete change it implies.
- **defer**: into the project's risks or follow-ups file.
- **reject**: with the rationale recorded so nothing re-argues it.

A finding that only yields speculation goes to a spike or a risk entry, not a disposition question. Don't relitigate what the user decided this session.

## 6. Apply

- **plan**: accepted findings edit the plan; deferred ones go to its risks section; rejected ones get a one-line decision entry naming why.
- **code**: never fix code from this skill. Accepted findings become new work items or follow-up entries; the user picks which.

## 7. Report

```
MULTI-LLM REVIEW — {target} ({mode})
Challengers: codex ✓ gemini ✓        Dropped in verification: {n}

| # | Source(s) | Sev | Target | Finding | Verdict | Disposition |
|---|-----------|-----|--------|---------|---------|-------------|
```

Add one calibration line. Both models approving with zero findings on every run means the prompts have gone soft. Zero survivors on this run is success.

## Never

- Run Codex with `--write`, or edit any file as a side effect of review. Plan edits happen only after the user dispositions.
- Present an unverified model finding as fact. Models hallucinate file paths, identifiers and constraints.
- Reopen a settled decision on a model's preference. Only a concrete named failure reopens one.
- Substitute this skill for the project's checkpoint reviews (`story-review`) or quality gate.
