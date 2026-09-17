{{header_by_disposition}}

{{rationale}}

{{optional_reference_link}}

## Variables

- `{{header_by_disposition}}`:
  - `fixed` → *(no reply needed; fix commit already references the comment)*
  - `invalid` → `Thanks for the review. We're marking this as not applicable — {{rationale}}`
  - `deferred` → `Thanks for the review. Agreed in principle, but deferring to a later sprint — {{rationale}}`
  - `blocked` → `We attempted this fix but it didn't clear our quality gate. Tracking as a follow-up — {{rationale}}`

- `{{rationale}}` — 1–3 sentences. Cite the relevant rule or doc when possible.

- `{{optional_reference_link}}` — include when a doc supports the disposition:
  - a rule doc from `rule_docs`, linked relative to the repo root
  - a spec doc under `docs_root`, e.g. `See [spec/security-model.md](../../docs/spec/security-model.md)`
  - `See [lessons.md](./lessons.md#review-NNN)` — sprint branches only; omit on a non-sprint branch (no `lessons.md`)
  - Omit entirely when no useful reference exists.

## Tone

- Factual, brief, non-defensive
- Do not apologize; do not argue
- Prefer pointing at a doc over explaining in prose

## Examples

### invalid

> Thanks for the review. We're marking this as not applicable — `x-forwarded-for` is already stripped at the edge and cannot reach this handler; the platform-supplied client-IP header is the only trusted source and we already use it.
>
> See [spec/runtime-topology.md](../../docs/spec/runtime-topology.md) §3.

### deferred

> Thanks for the review. Agreed in principle, but deferring to a later sprint — the cleanup sweeper needs a shared scheduler abstraction we haven't built yet; tracking in sprint 031.
>
> See [lessons.md#review-018](./lessons.md#review-018).

### blocked

> We attempted this fix but it didn't clear our quality gate — the proposed schema change breaks an existing migration. Tracking as a follow-up in sprint 031; open to alternate approaches.
