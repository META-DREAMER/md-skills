# md-skills

Claude Code / Codex skill bundles, published as plugins so they can be turned on
**per repository** instead of living in everyone's global config.

Both harnesses read the same manifests, so one install command each:

```bash
# Claude Code
claude plugin marketplace add META-DREAMER/md-skills
claude plugin install design-eng@md-skills

# Codex
codex plugin marketplace add https://github.com/META-DREAMER/md-skills
codex plugin add design-eng@md-skills
```

## Turning a bundle on for one repo

Commit these and everyone who clones the repo gets the same skills.

`.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "md-skills": { "source": { "source": "github", "repo": "META-DREAMER/md-skills" } }
  },
  "enabledPlugins": { "design-eng@md-skills": true }
}
```

`.codex/config.toml`:

```toml
[marketplaces.md-skills]
source_type = "git"
source = "https://github.com/META-DREAMER/md-skills"

[plugins."design-eng@md-skills"]
enabled = true
```

Codex only reads a repo's `.codex/config.toml` once the project is **trusted**,
so run `codex` in the repo and accept the trust prompt first.

Setting a bundle to `false` in a repo turns it off even when it is enabled
globally — project config outranks user config in both harnesses.

## Bundles

| Bundle | What it's for | Skills |
| --- | --- | --- |
| [`design-eng`](plugins/design-eng) | UI polish, motion and component design — Emil Kowalski's skills plus our own design-engineering set. | 14 |
| [`cloudflare`](plugins/cloudflare) | Cloudflare Workers, Durable Objects, Wrangler and friends, pinned to one upstream commit. | 8 |
| [`ralph`](plugins/ralph) | The Ralph sprint loop — plan, build, polish, review sprint stories autonomously. | 6 |
| [`review`](plugins/review) | Code review workflows — CodeRabbit, multi-model review, and TDD. | 3 |

## Generated

This repository is a build artifact of
[META-DREAMER/agent-infra](https://github.com/META-DREAMER/agent-infra), built
from `6ec09ce` by `claude/bin/build-marketplace.sh`. Skills authored there
live in `claude/skills/`; third-party ones are vendored at a pinned commit and
redistributed here under their own licences (see each bundle's `licenses/`).

**Pull requests against this repo will be overwritten by the next build** — open
them against `agent-infra` instead.
