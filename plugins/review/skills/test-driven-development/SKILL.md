---
name: test-driven-development
description: TDD methodology for writing reliable, well-tested code. Use BEFORE implementation to write failing tests, and AFTER to verify coverage. Triggers on "write tests", "add tests", "TDD", or when starting/completing features.
---

# Test-Driven Development

The doctrine here is stack-agnostic. A project may add its own overlay skill of the same name naming its layers, runners, and colocation rules; load that too when it exists, and let it win on mechanics.

## Red first

The failing test must fail for the **right reason** — a wrong value, not a missing import or a typo in the test name. Watch it fail, then write the implementation. A test you never saw red proves nothing.

**Always test-first for:**

- Anything where a bug costs money, data, or access: payments, on-chain logic, auth, permissions.
- State machines and lifecycle transitions.
- Pure business logic — calculations, encoding, parsing, validation.
- Request handlers: auth, input validation, response shape.

**Test-after is acceptable for:** UI prototypes, one-off scripts, exploratory spikes.

## Verify on the surface that's broken

This is the most expensive lesson to relearn. A stubbed suite, a desktop browser, a headless run, and a green CI can all pass while the real surface fails — the deployed build, mobile Safari, the actual database, the live network. When a test and reality disagree, reality is right and the test is measuring something else.

Corollary: a bundle-time or runtime failure outranks a green test. If the test harness injects compatibility the shipped runtime doesn't have, the test is testing a different program.

## Pick the tool for the layer

| Layer | Approach |
|-------|----------|
| Pure logic | Unit tests. No mocks needed; if you need one, the logic isn't pure. |
| I/O adapters (DB, network, SDK clients) | Integration test against the real backend when practical; otherwise mock one narrow SDK function. Never hand-roll a fake of a query builder, client, or fluent API. |
| State machines | Extract the pure transition decision where possible; test orchestration through the public runtime seam. |
| Background ticks / schedulers | Drive through the public entrypoint the platform itself calls, rescheduling from durable state rather than in-memory timers. |
| HTTP handlers | Factory-with-deps: the test passes deps to the factory and drives the real routing and middleware. |

**Test seams are an escape hatch, not architecture.** Prefer the seam production callers already use. When a framework owns construction and leaves no constructor or factory seam, give it the smallest one possible: a single setter plus a read that returns the deps or null. Don't grow a production setter, a `has*` probe, or boot-time wiring to satisfy a test.

**Colocation:** keep test files next to the source they test, in whatever convention the project already uses.

## Where tests earn their keep

Integration tests through the seam real callers use carry most of the confidence: handlers with their middleware, state machines with real storage, components with the network faked at the boundary. Unit tests cover pure calculation and encoding. End-to-end covers only critical happy paths, because it is the slowest and flakiest tier. Follow the project's own ratios when it states them.

## Anti-patterns

- **Testing erased types and cast bypasses.** Don't cast an invalid value (`"nonsense" as Address`, `as any`) to test a strictly typed internal function. Trust compile-time types internally; trust schema parsing at external boundaries.
- **Trivial framework reflection.** Asserting on ORM column metadata or framework internals tests the framework, not your code.
- **Standard-library parsing duplication.** Testing that a `min(0).max(7)` schema rejects `99` only proves the schema library works. Test custom transforms, refinements, and regexes only.
- **Fragile length pins.** Don't assert static lengths of enums or tuple lists; exhaustive switch checking handles this without a magic number.
- **Hand-rolled third-party payloads.** Never test a parser boundary against an object you *guessed* matches an external API. Use a verbatim recorded capture or an integration test against the real backend. Sanitize the capture before committing it — no secrets, key material, or PII — keeping the production shape intact.
- **Fake-around-fake.** Mocking a library's builder chain to test trivial mapping. Use the real backend; same cost, real coverage.
- **Export-shape tests.** `expect(typeof mod.setX).toBe("function")` proves nothing. Delete them.
- **Production setters for tests.** Don't grow a public production API to satisfy a mock.
- **Reconstructing state inside the assertion.** Verify against what was persisted, not by re-running the setup logic with placeholder inputs.
- **Wide dep bags.** A `Deps` interface holds the external effects the workflow must call, not every helper. If it keeps growing, extract pure decisions or move coverage to an integration test.
- **Call-choreography tests.** Asserting every mock call makes refactors expensive. Assert durable outcomes instead: persisted rows, emitted result, request payload, status transition.
- **Duplicated snapshot types.** Reuse the type the source module exports instead of mirroring its shape in a test seam.
- **Tests that re-verify the compiler.** If the type system already guarantees it, the test is noise — delete it.

## Reference

- [Fixtures](references/fixtures.md) — determinism, minimalism, naming, factories.
