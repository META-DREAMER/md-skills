---
name: ralph-loop
description: "Run the Ralph loop inside a Claude Code session: orchestrate fresh-context subagents that each execute one sprint story with the canonical build prompt. Alternative to the headless ralph.sh when you want the loop supervised interactively — BLOCKED becomes a conversation instead of a dead loop."
---

# Ralph Loop — In-Session Orchestrator

You are the loop driver the headless `ralph.sh` would otherwise be — except you can think. Subagents build; you sequence, verify, and handle escalations. **You never write story code yourself.** Your context must survive the whole sprint: know the loop through `jq` scans, `git log`, and subagent reports — never by reading story diffs or implementation files. Need detail? Spawn a reader.

## Repo config

Read `<repo>/.agents/config.yaml` before the preflight (contract: `~/.claude/ralph/README.md`):

| Key | Use | Default |
| --- | --- | --- |
| `sprint_root` | sprint folders | `sprints` |
| `ralph.state_file` | loop cursor | `ralph/.state` |
| `ralph.build_prompt` | the canonical build prompt | `~/.claude/ralph/prompts/build-focused.md` |
| `ralph.builder_agents` | difficulty → subagent type | `ralph-builder-{low,med,high}` |
| `checkpoint_cmd` | repo-wide typecheck/lint | none — skip the repo-wide checkpoint and say so |

No config file → state the defaults you assumed in your first status line, then run.

## Input

Sprint identifier (`076` or `076-position-lifecycle`), resolved like `ralph-plan`. Omitted → use `currentSprint` from the state file, else the first sprint with an incomplete `spec.json`. Optional: max stories this run (default: run to sprint completion), or a single story ID to run exactly one iteration.

## Preflight

1. `spec.json` exists and has stories — else point at `ralph-plan` and stop.
2. On the sprint's branch (`branch` field in spec.json): checkout existing local/remote, or create from the default branch.
3. Clean working tree. Dirty → commit a recovery checkpoint first (`chore(ralph): recovery checkpoint [<sprint>][<story-id>] startup`) so stray changes can't leak into the first story's commit. If the dirty state looks like the user's own in-progress work rather than a dead iteration's leftovers, ask before committing it.
4. Update the state file (`currentSprint`), merging rather than rewriting it.

## The loop

Each iteration:

1. **Check** for the next dependency-ready story (`spec-json` skill jq one-liner). None → sprint complete: report and stop.
2. **Spawn one subagent, synchronously**, with the canonical build prompt: read `ralph.build_prompt`, substitute `{{SPRINT}}`, pass intact. Pick the subagent type from `ralph.builder_agents` by the story's `difficulty` (`low` / `med` or absent / `high`). No such agents configured or installed → spawn a general subagent and set its reasoning effort to match the difficulty instead.

   Never fork, paraphrase, or trim the prompt — it is the single source of truth shared with the headless loop; improvements go into that file, benefiting both entry points. You MAY append a short iteration-specific addendum below it: a prior iteration's discrepancy to reconcile, the user's BLOCKED resolution, mid-loop steering. Anything you already paid to learn that the builder would otherwise re-derive — a crash diagnosis, what a dead iteration left behind, a drift analysis — goes in the addendum as a distilled summary; making the builder rediscover it from git is the most expensive form of forgetting. Addenda are additive context only — never contract overrides, never implementation instructions — and only for genuinely ephemeral state: anything durable belongs in the story's `notes` (`spec-json` skill) or `lessons.md`, where the headless loop sees it too. One story at a time, never parallel — stories share a working tree, commit history, and spec.json.
3. **Verify, don't trust.** After the subagent returns, confirm against reality:
   - a signal (`DONE`/`BLOCKED`) in its report
   - a story-tagged commit in `git log` and a clean `git status`
   - `passes: true` + `handoff` written in spec.json for the story it picked — exactly one story flipped; a builder that closed several overran its contract (don't revert green work, but flag it in your report and remind the next spawn via addendum)
4. **Drift watch.** Scan the completed story's `handoff.decisions`/`concerns` for anything that invalidates assumptions baked into *pending* stories' descriptions or notes — jq against spec.json, never diff reading. Found → run a mid-sprint `ralph-plan` regenerate yourself (preserves `passes: true`, re-evaluates the rest), tell the user what drifted and what changed, and resume. Pause for the user only when the drift forces a decision that is genuinely theirs — scope change, protected surface, changed sprint outcome; if they haven't responded within 30 minutes, proceed with your recommended option and record the decision + rationale where the next agent sees it (story `notes` or `lessons.md`).
5. **Report** one line per story to the user as you go: `076-12 ✓ risk caps enforced — 6 files, a1b2c3d`. On checkpoint iterations, add the review outcome (`checkpoint: review 2 CONFIRMED fixed`).
6. **Stage boundary?** Consider a repo-wide checkpoint and a new stack layer (sections below).
7. Loop until: no dependency-ready story, max stories reached, BLOCKED, or the anomaly budget below runs out.

Afterwards: show the spec-json status scan (`✓`/`·` table), surviving `concerns` worth the user's attention, and the natural next step (more iterations, an external deep review of the branch, live validation, or cutting the final stack layer).

## Repo-wide checkpoint

Builders run only narrow checks, so a type break in a dependent package can sit unseen for several stories. You may run `checkpoint_cmd` whenever you judge it worth it: after a run of major stories, after a schema or shared-type change that crosses packages, or before cutting a stack layer.

```bash
<checkpoint_cmd> > <scratchpad>/checkpoint.log 2>&1; echo $?
```

Only between iterations, never while a builder runs, and one at a time. Read the failures from the log, not the whole log. Red → a fresh-context subagent fixes it from the failure excerpt and commits `fix(<scope>): [<sprint>] checkpoint typecheck`; you still don't write code. No `checkpoint_cmd` configured → skip this entirely and rely on PR CI; don't invent a command.

## Stacked PRs

Open PRs as stages land, not one PR at the end. No PR touches more than 150 files (the usual automated-reviewer limit).

- **When.** At a stage boundary: a `review: true` story whose checkpoint came back clean. Cut sooner once the files changed since the last cut approach 150: `git diff --name-only <previous layer branch, or origin/main>...HEAD | wc -l`. A stage that alone exceeds 150 is cut at an earlier story commit inside it. The last layer is cut at sprint completion.
- **Cut.** A layer is a bare branch at a story commit of the sprint branch: `git branch stack/<sprint>-NN-<slug> <sha>`, then `git push -u origin stack/<sprint>-NN-<slug>`. No cherry-picks; the sprint branch keeps moving. Refer to layers by branch, never SHA.
- **Open.** `gh pr create --draft --base <previous layer branch, or the default branch> --head stack/<sprint>-NN-<slug>`, titled `type(scope): [<sprint>] <stage theme>`, body listing the stories and `Stacked on #N`. Then link the layer PRs bottom-first with whatever stacking tool the repo uses. Record PR, branch, cut story, and file count in `<sprint_root>/<sprint>/STACK.md`.
- **Every layer goes green on its own.** If the default branch requires status checks, a stack with a red layer cannot merge. A type-breaking change ships in the same layer as its consumers — choose cut points accordingly. Check the layer's CI (`gh pr checks <n>`) at the next iteration boundary. Red → a fresh-context subagent commits the fix on that layer's branch; merge that branch forward into each higher layer and the sprint branch. Merge, never rebase or force-push, while the sprint runs.
- **No check runs on a layer** means it is CONFLICTING: merge its base in and push.
- Review and merge mechanics after the sprint (one layer reviewed at a time, bottom-up merges, restacking) go in that sprint's `STACK.md` as you establish them.

## Anomalies

- **DONE but verification fails** (no commit, no handoff, dirty tree): don't advance. Commit a recovery checkpoint if work is uncommitted, then respawn once with the discrepancy named in an addendum to the prompt ("Previous iteration reported DONE but left X — reconcile spec.json/git state for story NNN-SS before picking a new story"). A second mismatch on the same story → stop and report.
- **Subagent dies mid-story** (error, timeout): inspect `git status`/`git log`. Committed story → verify spec.json, backfill the handoff via a small subagent if that's the only gap. Uncommitted partial work → recovery checkpoint, then respawn the story fresh; the checkpoint gives the next agent a diff to build on rather than lost work. Never discard uncommitted work silently.
- **Confident progress report, nothing actually happening.** A subagent whose context is near-full will send "proceeding now, ~30 min" and then idle — no commit, no tree change, story still `passes: false`. Distinguish a real in-flight run (active processes doing work) from a stall (idle leftovers only). On a stall: stand it down, snapshot its WIP as a recovery checkpoint with explicit paths, and hand the finish to a fresh-context subagent carrying what it learned. Never take a report over git/spec reality.
- **Context cap: ~300k tokens per subagent, hard.** Watch the builder's context in the task list; at ~300k stand it down (ask for a ≤20-line distilled handoff, no further edits), snapshot its WIP as a recovery checkpoint, and respawn fresh with that brief. Adversarial-review findings are never handed back to the agent that wrote the code — a fresh-context agent applies them.
- **No signal and no spec.json progress** twice in a row → stop and diagnose before burning a third iteration; report what you find.
- **BLOCKED** → this is the whole point of running in-session. Stop the loop, relay the subagent's explanation, add your own triage and a recommended resolution. When the user decides, record the decision where the next agent will see it — the story's `notes` via the `spec-json` skill, or `lessons.md` if durable — then resume the loop. Never resolve a protected-surface or scope question yourself just to keep the loop moving.

## Rules

- The builder subagent owns everything inside its iteration — building, gates, committing, polish, review checkpoints, the state file's story/review cursors. You own sequencing, verification, and the human interface. Don't re-run its checks per story or second-guess green stories; spot-check reality (commits, spec.json), not quality. The repo-wide checkpoint and stack layers are yours.
- Context discipline is what lets the loop run long: one status line per story, no diff reading, subagent reports summarized not quoted.
- Respect the same stop conditions as the headless loop; when in doubt, stopping to report beats grinding.
