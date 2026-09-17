# Invocation Mechanics

## Codex (openai-codex plugin — NOT an MCP)

**Detect:** the `CLAUDE_PLUGIN_ROOT` env var, else glob `~/.claude/plugins/cache/openai-codex/codex/*/`. The companion script is `{plugin_root}/scripts/codex-companion.mjs`. Not found → `codex skipped: plugin not installed`.

**Plan mode** — a plan document is not a git diff, so the plugin's `adversarial-review` (diff-scoped) does not apply. Use `task`:

```bash
node "{codex_companion}" task "$(cat {scratchpad}/codex-plan-prompt.md)"
```

- Write the prompt to a scratchpad file first (shell escaping), then run the companion **foreground** inside a `Bash` call with `run_in_background: true` — the result arrives as the process output when it completes, with no job polling. Have it write its answer to a scratchpad file too, so a dropped connection doesn't lose the run.
- Ceiling: 20 minutes plan mode (Codex reads the repo to verify claims — a deep run is legitimate, don't punish slow reasoning), 10 minutes code mode. The ceiling keeps the session moving, not the model honest: at the ceiling, ask the user whether to keep waiting. Non-zero exit → warn and continue without Codex. Either way, verify the other challenger's findings while waiting.
- Read-only by default — **never pass `--write`** from this skill.
- Leave `--model` and `--effort` unset (plugin defaults, per its `codex-cli-runtime` contract).
- Codex's read-only sandbox still has repo access. That is its edge over Gemini: the prompt tells it to verify the plan's claims against actual code, so don't inline file contents it can read itself.

**Code mode** — use the plugin's ready-made contract (its own adversarial prompt and output schema):

```bash
node "{codex_companion}" adversarial-review --wait --base <base-branch> --scope branch "<focus text from prompts.md>"
```

Same foreground-in-background-Bash handling. Output conforms to `{plugin_root}/schemas/review-output.schema.json`: `{ verdict: "approve"|"needs-attention", summary, findings: [{ severity, title, body, file, line_start, line_end, confidence, recommendation }], next_steps }`.

## Gemini (gemini-cli MCP)

**Detect/load:** one ToolSearch call — `select:mcp__gemini-cli__ask-gemini,mcp__gemini-cli__fetch-chunk,mcp__gemini-cli__ping` — then `ping`. Unavailable → `gemini skipped: mcp not connected`.

**Both modes** — a single `ask-gemini` call:

- `prompt`: the mode's template from `prompts.md`. Include dossier files with `@` syntax — Gemini cannot fetch anything itself, so the dossier must be complete. The MCP **refuses `@` paths outside the project directory** (verified live), so everything referenced must live in-repo. For the code-mode diff, write `git diff <base>...HEAD` to a gitignored in-repo scratch path and reference it relatively; delete it after the run. If an `@` include fails to resolve, fall back to inlining that content.
- `model`: unset. It defaults to the strongest tier; do not downgrade to a flash model for review work.
- `changeMode`: `false`, always. Structured-edit mode invites the model to rewrite the artifact; this skill wants findings, not edits.
- Long responses arrive chunked: follow the returned `chunkCacheKey` with `fetch-chunk` until complete before parsing.
- The backend has changed hands before and may ignore the JSON contract, returning markdown findings, sometimes with leaked system-prompt noise at the top. Parse leniently — findings in any structured form count. Strip the noise. Retry only when findings are absent or garbled, not merely non-JSON.

## Shared failure handling

- Exactly one challenger available → proceed single-model and say so in the report header.
- A challenger returns unparseable output → one retry restating the output contract; still broken → treat as skipped, warn, continue.
- Both skipped or failed → stop. Report why and suggest the fixes (`/codex:setup`, reconnect the gemini MCP). Do not fall back to self-review: the value of this skill is models that aren't you.
- Long or flaky runs → instruct each challenger to write results to a scratchpad file and read the file rather than relying on the response surviving.
