---
name: drop-artifact
description: Generate a shareable HTML artifact (roadmap, one-pager, design memo, status update, mockup, or a small interactive app) and publish it to a short shareable URL via the `drop` Cloudflare Worker. Use when asked to make a shareable visual/HTML page, turn a doc or plan into a link, or "drop" something for the team to view.
---

# drop-artifact

Build a self-contained HTML artifact, publish it, and hand back the URL.

**Follow the canonical playbook** at `https://drop.md-7c2.workers.dev/SKILL.md`. It covers artifact shapes, the quality bar, the per-content-type playbook, where to save the source, and publishing through the HTTP API. Replace `{{ORIGIN}}` with `https://drop.md-7c2.workers.dev`.

**Start from the project's house style** when it defines one (design tokens, fonts, layout in an existing artifact or design doc) and the subject is that project. Depart when a different subject wants a different look.
