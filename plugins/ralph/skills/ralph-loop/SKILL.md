---
name: ralph-loop
description: "Run the Ralph loop inside a Claude Code session: orchestrate fresh-context subagents that each execute one sprint story with the canonical build prompt. Use instead of the headless ralph.sh when you want the loop supervised interactively, so a BLOCKED story becomes a conversation instead of a dead loop."
---

# Ralph Loop

You replace the headless `ralph.sh` as loop driver. Subagents build; you sequence, verify, and handle escalations. **You never write story code.** Your context has to last the whole sprint, so you learn state from `jq` scans, `git log` and subagent reports, never from story diffs or implementation files. Need detail? Spawn a reader.

## Repo facts

**Repo facts.** Resolve each fact below in order: its `.agents/config.yaml` key; what the project's `CLAUDE.md`, `AGENTS.md` or README names; the conventional places listed. Relative paths resolve from the repo root. A fact you only read and cannot find is skipped with a note; one you must write is created at the default shown. Never guess a command. Say what you resolved, and from where, in your first status line.

| Fact | Key | Look for | If none |
| --- | --- | --- | --- |
| Sprint folders | `sprint_root` | an existing `sprints/` | create `sprints/` |
| Repo-wide gate | `checkpoint_cmd` | `CLAUDE.md`, the CI workflow's gate step | skip it and say so |
| Loop cursor | `ralph.state_file` | an existing `ralph/.state` | create `ralph/.state` |
| Build prompt | `ralph.build_prompt` | `ralph/prompts/build-focused.md` in the repo | `~/.claude/ralph/prompts/build-focused.md` |
| Constitution | `ralph.constitution` | `ralph/constitution.md` in the repo | `~/.claude/ralph/constitution.md` |
| Builder agents | `ralph.builder_agents` | installed `ralph-builder-{low,med,high}` agents | a general subagent with effort matched to difficulty |

## Input

Sprint identifier (`076` or `076-position-lifecycle`), resolved like `ralph-plan`. Omitted → `currentSprint` from the state file, else the first sprint with an incomplete `spec.json`. Optional: max stories this run (default: to completion), or one story ID for a single iteration.

## Preflight

1. `spec.json` exists and has stories, else point at `ralph-plan` and stop.
2. Check out the sprint's branch (`branch` in spec.json), or create it from the default branch.
3. Dirty tree → commit a recovery checkpoint (`chore(ralph): recovery checkpoint [<sprint>][<story-id>] startup`) so stray changes can't leak into the first story. If it looks like the user's own work rather than a dead iteration's leftovers, ask first.
4. Set `currentSprint` in the state file, merging rather than rewriting.

## The loop

Each iteration:

1. **Check** for the next dependency-ready story (`spec-json` jq one-liner). None → sprint complete: report and stop.
2. **Spawn one builder, synchronously.** Read `ralph.build_prompt`, substitute `{{SPRINT}}`, and pass it intact. Pick the subagent type from `ralph.builder_agents` by the story's `difficulty` (`low`, `med` or absent, `high`).

   Never fork, paraphrase or trim the prompt. The headless loop uses the same file, so improvements go into it. You may append a short addendum for ephemeral state only: a discrepancy to reconcile, the user's BLOCKED resolution, mid-loop steering, or a distilled summary of something you already paid to learn (a crash diagnosis, what a dead iteration left, a drift analysis). Addenda add context; they never override the contract or dictate implementation. Anything durable goes in the story's `notes` or `lessons.md`, where the headless loop sees it too. One story at a time: stories share a tree, history and spec.json.
3. **Verify against reality:** a `DONE`/`BLOCKED` signal in the report; a story-tagged commit and a clean `git status`; `passes: true` plus `handoff` for exactly the story it picked. A builder that closed several stories overran its contract: keep the green work, flag it, and remind the next spawn in the addendum.
4. **Drift watch.** Scan the finished story's `handoff.decisions`/`concerns` (jq, not diffs) for anything that invalidates pending stories' descriptions or notes. Found → run a `ralph-plan` regenerate yourself, tell the user what drifted and what changed, and resume. Pause only when the drift forces a decision that is the user's (scope, protected surface, sprint outcome). No answer within 30 minutes → proceed with your recommendation and record it in the story's `notes` or `lessons.md`.
5. **Report** one line per story: `076-12 ✓ risk caps enforced — 6 files, a1b2c3d`. On checkpoint stories add the review outcome (`checkpoint: review 2 CONFIRMED fixed`).
6. **Stage boundary?** Consider a repo-wide checkpoint and a stack layer (below).
7. Stop on: no ready story, max stories reached, BLOCKED, or the anomaly budget running out.

Afterwards: the spec-json status scan, surviving `concerns` worth the user's attention, and the next step (more iterations, an external deep review, live validation, or the final stack layer).

## Repo-wide checkpoint

Builders run only narrow checks, so a break in a dependent package can sit unseen for several stories. Run `checkpoint_cmd` when you judge it worth it: after a run of major stories, after a cross-package schema or shared-type change, or before cutting a stack layer. If `rt` is on PATH, offload it:

```bash
rt run -- "<checkpoint_cmd>" > <scratchpad>/checkpoint.log 2>&1; echo $?   # without rt: run <checkpoint_cmd> directly
```

Only between iterations and one at a time, remote runs included: a builder and a remote checkpoint share one remote worktree. Read the failures, not the whole log. Red → a fresh-context subagent fixes it from the failure excerpt and commits `fix(<scope>): [<sprint>] checkpoint typecheck`. No `checkpoint_cmd` → rely on PR CI.

## Stacked PRs

Open PRs as stages land. No PR touches more than 150 files (the usual automated-reviewer limit). The repo's `rule_docs` may name its stacking tool, required checks and review mechanics.

- **When.** At a `review: true` story whose checkpoint came back clean, or sooner once `git diff --name-only <previous layer, or origin/<default>>...HEAD | wc -l` approaches 150. A stage over 150 on its own is cut at an earlier story commit. The last layer is cut at sprint completion.
- **Cut.** `git branch stack/<sprint>-NN-<slug> <sha>` at a story commit, then `git push -u origin stack/<sprint>-NN-<slug>`. No cherry-picks; the sprint branch keeps moving. Refer to layers by branch, never SHA.
- **Open.** `gh pr create --draft --base <previous layer, or the default branch> --head stack/<sprint>-NN-<slug>`, titled `type(scope): [<sprint>] <stage theme>`, body listing the stories and `Stacked on #N`. Link the layers bottom-first with the repo's stacking tool. Record PR, branch, cut story and file count in `<sprint_root>/<sprint>/STACK.md`, plus review and merge mechanics as you establish them.
- **Every layer goes green on its own.** With required status checks, one red layer blocks the stack. A type-breaking change ships in the same layer as its consumers. Check each layer's CI (`gh pr checks <n>`) at the next iteration boundary. Red → a fresh-context subagent fixes it on that layer's branch; merge it forward into each higher layer and the sprint branch. Merge, never rebase or force-push, while the sprint runs.
- **No check runs on a layer** means it is CONFLICTING: merge its base in and push.

## Anomalies

- **DONE but verification fails** (no commit, no handoff, dirty tree): don't advance. Commit a recovery checkpoint if work is uncommitted, then respawn once with the discrepancy in the addendum. A second mismatch on the same story → stop and report.
- **Builder dies mid-story.** Committed → verify spec.json and backfill a missing handoff with a small subagent. Uncommitted → recovery checkpoint, then respawn the story fresh. Never discard uncommitted work silently.
- **Confident report, nothing happening.** A builder near a full context says "proceeding now" and idles: no commit, no tree change, still `passes: false`. Tell a real in-flight run (processes doing work) from a stall. On a stall: stand it down, snapshot its WIP as a recovery checkpoint with explicit paths, and hand the finish to a fresh subagent with what it learned.
- **Context cap: about 300k tokens per builder.** At the cap, ask for a distilled handoff of 20 lines or fewer and no further edits, snapshot its WIP, and respawn fresh with that brief. Review findings never go back to the agent that wrote the code; a fresh agent applies them.
- **No signal and no spec.json progress** twice in a row → stop and diagnose before a third iteration.
- **BLOCKED.** Stop, relay the builder's explanation with your triage and a recommended resolution. Record the user's decision in the story's `notes` (or `lessons.md` if durable), then resume. Never resolve a protected-surface or scope question yourself to keep the loop moving.

## Rules

- The builder owns everything inside its iteration: building, gates, commits, polish, review checkpoints, and the state file's story and review cursors. You own sequencing, verification, the repo-wide checkpoint, stack layers and the human interface. Spot-check reality (commits, spec.json), not quality.
- One status line per story, no diff reading, subagent reports summarized, not quoted.
- Honour the stop conditions in `ralph.constitution`. When in doubt, stop and report.
