---
name: drop-artifact
description: Generate a shareable HTML artifact (roadmap, one-pager, design memo, status update, mockup, or a small interactive app) and publish it to a short shareable URL via the `drop` Cloudflare Worker. Use when asked to make a shareable visual/HTML page, turn a doc or plan into a link, or "drop" something for the team to view.
---

# drop-artifact — generate & publish a shareable artifact

Build a self-contained HTML artifact, publish it, hand back the URL.

**Build + publish — follow the canonical playbook:** fetch `https://drop.md-7c2.workers.dev/SKILL.md`. It covers the artifact shapes, the quality bar, the per-content-type category playbook, where to save the source, and publishing via the HTTP API. Wherever it shows `{{ORIGIN}}`, use `https://drop.md-7c2.workers.dev`.

**House style:** if the project defines one (design tokens, fonts, layout in an existing artifact or design doc), start from it and match it when the subject is that project. Depart when a different subject wants a different look.
