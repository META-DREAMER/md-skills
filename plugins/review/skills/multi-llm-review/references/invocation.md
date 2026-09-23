# Invocation Mechanics

## Codex (openai-codex plugin, not an MCP)

**Detect:** the `CLAUDE_PLUGIN_ROOT` env var, else glob `~/.claude/plugins/cache/openai-codex/codex/*/`. The companion script is `{plugin_root}/scripts/codex-companion.mjs`. Not found: `codex skipped: plugin not installed`.

**Plan mode.** A plan is not a git diff, so the diff-scoped `adversarial-review` does not apply. Use `task`:

```bash
node "{codex_companion}" task "$(cat {scratchpad}/codex-plan-prompt.md)"
```

- Write the prompt to a scratchpad file first (shell escaping). Run the companion in the foreground inside a `Bash` call with `run_in_background: true`; the result arrives as process output, with no job polling. Have it also write its answer to a scratchpad file so a dropped connection doesn't lose the run.
- The ceiling (20 minutes plan, 10 code) keeps the session moving; a deep run is legitimate. At the ceiling, ask the user. Non-zero exit: warn and continue without Codex.
- Never pass `--write`. Leave `--model` and `--effort` unset (plugin defaults, per its `codex-cli-runtime` contract).
- Codex's read-only sandbox has repo access; that is its edge. Don't inline file contents it can read itself.

**Code mode.** Use the plugin's own adversarial prompt and output schema:

```bash
node "{codex_companion}" adversarial-review --wait --base <base-branch> --scope branch "<focus text from prompts.md>"
```

Same foreground-in-background handling. Output follows `{plugin_root}/schemas/review-output.schema.json`: `{ verdict: "approve"|"needs-attention", summary, findings: [{ severity, title, body, file, line_start, line_end, confidence, recommendation }], next_steps }`.

## Gemini (gemini-cli MCP)

**Detect:** one ToolSearch call, `select:mcp__gemini-cli__ask-gemini,mcp__gemini-cli__fetch-chunk,mcp__gemini-cli__ping`, then `ping`. Unavailable: `gemini skipped: mcp not connected`.

**Both modes** use a single `ask-gemini` call:

- `prompt`: the mode's template, with dossier files included by `@` path. Gemini cannot fetch anything, so the dossier must be complete. **The MCP refuses `@` paths outside the project directory.** Write the code-mode diff to a gitignored in-repo scratch path, reference it relatively, and delete it after the run. If an `@` include fails to resolve, inline that content.
- `model`: unset. It defaults to the strongest tier; never downgrade to a flash model for review.
- `changeMode`: always `false`. Edit mode invites the model to rewrite the artifact.
- Long responses arrive chunked: follow `chunkCacheKey` with `fetch-chunk` until complete.
- **The backend may ignore the JSON contract**, returning markdown findings, sometimes with leaked system-prompt noise at the top. Parse leniently and strip the noise. Retry only when findings are absent or garbled, not merely non-JSON.

## Failure handling

- One challenger available: proceed single-model and say so in the report header.
- Unparseable output: one retry restating the output contract; still broken, treat as skipped.
- Both skipped or failed: stop, report why, and suggest fixes (`/codex:setup`, reconnect the gemini MCP). Never fall back to self-review.
