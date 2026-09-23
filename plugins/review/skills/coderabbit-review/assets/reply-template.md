{{header_by_disposition}}

{{rationale}}

{{optional_reference_link}}

## Variables

- `{{header_by_disposition}}`:
  - `fixed`: no reply; the fix commit references the comment.
  - `invalid`: `Thanks for the review. We're marking this as not applicable — {{rationale}}`
  - `deferred`: `Thanks for the review. Agreed in principle, but deferring to a later sprint — {{rationale}}`
  - `blocked`: `We attempted this fix but it didn't clear our quality gate. Tracking as a follow-up — {{rationale}}`
- `{{rationale}}`: 1 to 3 sentences, citing the rule or doc when possible.
- `{{optional_reference_link}}`: a rule doc from `rule_docs` or a spec doc under `docs_root`, linked relative to the repo root; on a sprint branch, `See [lessons.md](./lessons.md#review-NNN)`. Omit when nothing useful exists.

Tone: factual, brief, non-defensive. Don't apologize or argue; point at a doc rather than explain in prose.

## Example (invalid)

> Thanks for the review. We're marking this as not applicable — `x-forwarded-for` is already stripped at the edge and cannot reach this handler; the platform-supplied client-IP header is the only trusted source and we already use it.
>
> See [spec/runtime-topology.md](../../docs/spec/runtime-topology.md) §3.
