---
name: spec-json
description: "Schema and commands to read and write to spec.json files. Load when reading or writing sprint state, or investigating how and why past features were implemented."
---

# spec-json

`<sprint_root>/<sprint>/spec.json` is the sprint's source of truth: stories, dependencies, completion state, and per-story `handoff` worklog. Files run 500–2000 lines — query with `jq` instead of reading whole.

`sprint_root` comes from `<repo>/.agents/config.yaml` (default `sprints`; full contract in `~/.claude/ralph/README.md`). The examples below write `sprints/` for readability — substitute the configured root.

Two modes of use:

- **Execution** — pick the next story, load context, mark complete, write handoff.
- **Investigation** — mine `handoff.decisions` and `lessons.md` to understand how and why a past feature was built. The worklog is the durable "why" behind shipped code.

## Schema

Top-level: `sprint`, `branch`, `generatedAt`, `mode` (`"normal" | "regenerate"`), `stories[]`, `interviewSummary { scopeChanges, risks[], decisions[] }`.

Story:

| Field                  | Type                                   | Notes                                     |
| ---------------------- | -------------------------------------- | ----------------------------------------- |
| `id`                   | `<NNN>-<XX>`                           | e.g. `041-17`                             |
| `title`, `description` | string                                 | scope                                     |
| `deps`                 | string[]                               | story IDs; only earlier IDs               |
| `acceptanceCriteria`   | string[]                               | each names its proving evidence; the quality gate is implicit, not a criterion (legacy sprints may open with `"Typecheck passes"`) |
| `testFirst`            | bool                                   | TDD flag                                  |
| `difficulty`           | `"low" \| "med" \| "high"`?            | absent = `med`; sets the implementing agent's reasoning effort + how prescriptive `notes` are; `high` adds one post-implementation adversarial review of the diff (see build prompt) |
| `review`               | bool?                                  | review checkpoint — completing this story triggers `story-review` over the cumulative diff since the last checkpoint |
| `passes`               | bool                                   | completion gate                           |
| `notes`                | string?                                | optional hints                            |
| `completedAt`          | ISO-8601?                              | set on completion                         |
| `handoff`              | `{ done[], decisions[], concerns[] }`? | set on completion                         |
| `filesChanged`         | number?                                | set on completion                         |

`handoff` is the durable per-story record: `done` = specific facts (files, counts, URLs), `decisions` = non-obvious choices + why, `concerns` = risks/edge cases. Story-local context lives here. Cross-sprint patterns go in `lessons.md`.

## jq Patterns

Pick the next dependency-ready story (or empty if sprint done):

```bash
jq -r '
  [.stories[] | select(.passes == true) | .id] as $p
  | first(.stories[] | select(.passes != true) | select((.deps // []) | all(. as $d | $p | index($d)))).id // empty
' sprints/<sprint>/spec.json
```

Single story by ID:

```bash
jq --arg id "<id>" 'first(.stories[] | select(.id == $id))' sprints/<sprint>/spec.json
```

Handoffs from a story's dependencies:

```bash
jq --arg id "<id>" '
  (.stories[] | select(.id == $id) | .deps // []) as $d
  | [.stories[] | select(.id as $s | $d | index($s)) | {id, title, handoff}]
' sprints/<sprint>/spec.json
```

Status scan (`✓`/`·` per story):

```bash
jq -r '.stories[] | "\(.id)\t\(if .passes then "✓" else "·" end)\t\(.title)"' sprints/<sprint>/spec.json
```

**Always edit via `jq … > tmp && mv`. Never rewrite spec.json with the `Write` tool or whole-file `Edit`s.** `jq` emits non-ASCII as literal UTF-8 bytes (e.g. em-dash `—`, section sign `§`); models often emit the same characters as `\uXXXX` JSON escapes instead. Both parse to the same JSON, but mixing them flips the encoding across commits and produces huge spurious diffs that ping-pong every iteration. `jq`'s default UTF-8 output is the canonical form — stay on it. If you must do a structural rewrite that's hard to express in `jq`, round-trip through `jq '.' file > file.tmp && mv file.tmp file` afterwards to renormalize.

Mark a story complete — atomic via `tmp && mv`, never redirect into the same file:

```bash
jq --arg id "<id>" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   --argjson handoff '{"done":[...],"decisions":[...],"concerns":[...]}' \
   --argjson files <N> \
   '.stories |= map(if .id == $id then . + {passes:true, completedAt:$now, handoff:$handoff, filesChanged:$files} else . end)' \
   sprints/<sprint>/spec.json > sprints/<sprint>/spec.json.tmp \
   && mv sprints/<sprint>/spec.json.tmp sprints/<sprint>/spec.json
```

Compose freely — these are starting points, not a closed set.

## Loop state file

The loop cursor lives at `ralph.state_file` (default `ralph/.state`): `currentSprint`, `lastStoryCompleted`, `lastUpdated`, `lastReviewedSha`. Update it the same way — merge with `jq`, never rewrite the whole file, or you drop the other lane's keys:

```bash
jq --arg story "<id>" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.lastStoryCompleted = $story | .lastUpdated = $now' \
   ralph/.state > ralph/.state.tmp && mv ralph/.state.tmp ralph/.state
```

## lessons.md

`<sprint_root>/<sprint>/lessons.md` is append-only durable knowledge for **future work in this sprint**. Default: add nothing. Add an entry only when it's durable, non-obvious, would save future rediscovery, and isn't already in `handoff` / code / architecture decisions / docs. If it only matters for the story you just finished, it belongs in `handoff`.

Append at the tail of `## Gotchas & Patterns`:

```markdown
### <short topic> — <STORY_ID>

<concise body>
```

Never edit or reorder existing entries. Graduate cross-sprint rules into the repo's `rule_docs` at sprint close.
