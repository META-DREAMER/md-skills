---
name: spec-json
description: "Schema and jq commands to read and write sprint spec.json files. Load when reading or writing sprint state, or investigating how and why past features were implemented."
---

# spec-json

`<sprint_root>/<sprint>/spec.json` is the sprint's source of truth: stories, dependencies, completion state and per-story `handoff`. Files run 500 to 2000 lines, so query with `jq` instead of reading them whole.

Two uses: **execution** (pick the next story, load context, mark complete, write handoff) and **investigation** (mine `handoff.decisions` and `lessons.md` for how and why a past feature was built).

## Repo facts

**Repo facts.** Resolve each fact below in order: its `.agents/config.yaml` key; what the project's `CLAUDE.md`, `AGENTS.md` or README names; the conventional places listed. Relative paths resolve from the repo root. A fact you only read and cannot find is skipped with a note; one you must write is created at the default shown. Never guess a command. Say what you resolved, and from where, in your first status line.

| Fact | Key | Look for | If none |
| --- | --- | --- | --- |
| Sprint folders | `sprint_root` | an existing `sprints/` | create `sprints/` |
| Rule docs | `rule_docs` | root and package `CLAUDE.md` / `AGENTS.md`, and the docs they route to | root `CLAUDE.md` or `AGENTS.md` |
| Loop cursor | `ralph.state_file` | an existing `ralph/.state` | create `ralph/.state` |

## Schema

Top-level: `sprint`, `branch`, `generatedAt`, `mode` (`"normal" | "regenerate"`), `stories[]`, `interviewSummary { scopeChanges, risks[], decisions[] }`.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | `<NNN>-<XX>` | e.g. `041-17` |
| `title`, `description` | string | scope |
| `deps` | string[] | earlier story IDs only |
| `acceptanceCriteria` | string[] | each names its evidence; the quality gate is implicit (legacy sprints may open with `"Typecheck passes"`) |
| `testFirst` | bool | TDD flag |
| `difficulty` | `"low" \| "med" \| "high"`? | absent = `med`; sets builder effort and how prescriptive `notes` are; `high` adds one adversarial review of the diff |
| `review` | bool? | checkpoint: `story-review` runs over the diff since the last checkpoint |
| `passes` | bool | completion gate |
| `notes` | string? | hints |
| `completedAt` | ISO-8601? | set on completion |
| `handoff` | `{ done[], decisions[], concerns[] }`? | set on completion |
| `filesChanged` | number? | set on completion |

`handoff`: `done` = specific facts (files, counts, URLs), `decisions` = non-obvious choices and why, `concerns` = risks and edge cases. Story-local context lives here; cross-story patterns go in `lessons.md`.

## jq patterns

Next dependency-ready story (empty when the sprint is done):

```bash
jq -r '
  [.stories[] | select(.passes == true) | .id] as $p
  | first(.stories[] | select(.passes != true) | select((.deps // []) | all(. as $d | $p | index($d)))).id // empty
' <sprint_root>/<sprint>/spec.json
```

One story:

```bash
jq --arg id "<id>" 'first(.stories[] | select(.id == $id))' <sprint_root>/<sprint>/spec.json
```

Handoffs of a story's dependencies:

```bash
jq --arg id "<id>" '
  (.stories[] | select(.id == $id) | .deps // []) as $d
  | [.stories[] | select(.id as $s | $d | index($s)) | {id, title, handoff}]
' <sprint_root>/<sprint>/spec.json
```

Status scan:

```bash
jq -r '.stories[] | "\(.id)\t\(if .passes then "✓" else "·" end)\t\(.title)"' <sprint_root>/<sprint>/spec.json
```

**Edit only via `jq … > tmp && mv`, never the `Write` tool or whole-file edits.** `jq` writes non-ASCII as literal UTF-8; models often write `\uXXXX` escapes. Mixing the two flips the encoding between commits and produces large spurious diffs. After a structural rewrite jq can't express, renormalize with `jq '.' file > file.tmp && mv file.tmp file`.

Mark a story complete (never redirect into the same file):

```bash
jq --arg id "<id>" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   --argjson handoff '{"done":[...],"decisions":[...],"concerns":[...]}' \
   --argjson files <N> \
   '.stories |= map(if .id == $id then . + {passes:true, completedAt:$now, handoff:$handoff, filesChanged:$files} else . end)' \
   <sprint_root>/<sprint>/spec.json > <sprint_root>/<sprint>/spec.json.tmp \
   && mv <sprint_root>/<sprint>/spec.json.tmp <sprint_root>/<sprint>/spec.json
```

## Loop state file

`ralph.state_file` holds `currentSprint`, `lastStoryCompleted`, `lastUpdated`, `lastReviewedSha`. Merge with `jq`; a whole-file rewrite drops the other lane's keys:

```bash
jq --arg story "<id>" --arg now "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   '.lastStoryCompleted = $story | .lastUpdated = $now' \
   <state_file> > <state_file>.tmp && mv <state_file>.tmp <state_file>
```

## lessons.md

`<sprint_root>/<sprint>/lessons.md` is append-only knowledge for future work in the sprint. Default: add nothing. Add an entry only when it is durable, non-obvious, saves rediscovery, and isn't already in `handoff`, code, architecture decisions or docs. Append at the tail of `## Gotchas & Patterns`:

```markdown
### <short topic> — <STORY_ID>

<concise body>
```

Never edit or reorder entries. Graduate cross-sprint rules into the `rule_docs` at sprint close.
