# Fixtures

- **Deterministic.** No `Date.now()`, `Math.random()` or auto-increment ids. Use fixed values.
- **Minimal.** Include only the fields the test cares about; let a factory fill the rest.
- **Explicit.** Each test shows what data it depends on. Avoid distant shared state.
- **Named `<entity>_<variant>`**: `position_active`, `position_underwater`, `user_funded`, `token_usdc`. Cover the variants that matter: funded and unfunded accounts, token decimals (18, 6, 8), pending, confirmed and failed transactions.

Use a factory when fixtures need computed fields:

```typescript
function makePosition(overrides?: Partial<Position>): Position {
  return { id: "pos-test", status: "active", healthFactor: 1.85, ...overrides };
}
```
