---
name: test-driven-development
description: TDD methodology for writing reliable, well-tested code. Use before implementation to write failing tests, and after to verify coverage. Triggers on "write tests", "add tests", "TDD", or when starting/completing features.
---

# Test-Driven Development

The doctrine is stack-agnostic. The project's `CLAUDE.md` and testing docs name its runners, tiers and colocation rules; they win on mechanics.

## Red first

**The failing test must fail for the right reason:** a wrong value, not a missing import or a typo. Watch it fail, then implement. A test you never saw red proves nothing.

Always test-first for:

- Anything where a bug costs money, data or access: payments, on-chain logic, auth, permissions.
- State machines and lifecycle transitions.
- Pure business logic: calculations, encoding, parsing, validation.
- Request handlers: auth, input validation, response shape.

Test-after is acceptable for UI prototypes, one-off scripts and exploratory spikes.

## Verify on the surface that's broken

**A stubbed suite, a desktop browser, a headless run and green CI can all pass while the real surface fails**: the deployed build, mobile Safari, the real database, the live network. When a test and reality disagree, the test is measuring something else.

A bundle-time or runtime failure outranks a green test. If the harness injects compatibility the shipped runtime lacks, the test is testing a different program.

## Pick the tool for the layer

| Layer | Approach |
|-------|----------|
| Pure logic | Unit tests. If you need a mock, the logic isn't pure. |
| I/O adapters (DB, network, SDK clients) | Integration test against the real backend when practical; otherwise mock one narrow SDK function. Never hand-roll a fake of a query builder, client or fluent API. |
| State machines | Extract the pure transition decision where possible; test orchestration through the public runtime seam. |
| Background ticks / schedulers | Drive through the entrypoint the platform itself calls, rescheduling from durable state, not in-memory timers. |
| HTTP handlers | Factory-with-deps: the test passes deps to the factory and drives the real routing and middleware. |

**Test seams are an escape hatch, not architecture.** Prefer the seam production callers already use. When a framework owns construction and leaves no constructor or factory seam, add the smallest one: a single setter plus a read that returns the deps or null. Never grow a production setter, a `has*` probe or boot-time wiring for a test.

Keep test files next to the source they test, in the project's existing convention.

## Where tests earn their keep

Integration tests through the seam real callers use carry most of the confidence: handlers with middleware, state machines with real storage, components with the network faked at the boundary. Unit tests cover pure calculation and encoding. End-to-end covers only critical happy paths; it is the slowest and flakiest tier. Follow the project's own ratios when it states them.

## Anti-patterns

- **Testing erased types.** Don't cast an invalid value (`"nonsense" as Address`, `as any`) to test a strictly typed internal function. Trust compile-time types internally and schema parsing at external boundaries.
- **Framework reflection.** Asserting on ORM column metadata or framework internals tests the framework.
- **Schema-library duplication.** Testing that `min(0).max(7)` rejects `99` proves the library works. Test only custom transforms, refinements and regexes.
- **Length pins.** Don't assert static lengths of enums or tuples; exhaustive switch checking covers it.
- **Hand-rolled third-party payloads.** Never test a parser against an object you guessed matches an external API. Use a verbatim recorded capture or an integration test. Sanitize the capture (no secrets, key material or PII) and keep the production shape.
- **Fake-around-fake.** Mocking a library's builder chain to test trivial mapping. Use the real backend; same cost, real coverage.
- **Export-shape tests.** `expect(typeof mod.setX).toBe("function")` proves nothing. Delete them.
- **Reconstructing state in the assertion.** Verify against what was persisted, not by re-running setup logic with placeholder inputs.
- **Wide dep bags.** A `Deps` interface holds the external effects the workflow must call, not every helper. If it keeps growing, extract pure decisions or move coverage to an integration test.
- **Call-choreography tests.** Asserting every mock call makes refactors expensive. Assert durable outcomes: persisted rows, emitted result, request payload, status transition.
- **Duplicated snapshot types.** Reuse the type the source module exports instead of mirroring it in a test seam.
- **Tests that re-verify the compiler.** Delete them.

Fixture rules: [references/fixtures.md](references/fixtures.md).
