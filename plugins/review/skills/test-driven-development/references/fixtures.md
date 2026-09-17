# Fixtures

## Principles

- **Deterministic** — No `Date.now()`, `Math.random()`, or auto-increment IDs in fixtures. Use fixed values.
- **Minimal** — Only include fields the test cares about. Let defaults/factories handle the rest.
- **Explicit** — Each test should make clear what data it depends on. Avoid distant shared state.

## Fixture Families

| Family | Examples |
|--------|----------|
| Accounts | `testUser`, `testAttacker`, funded/unfunded wallets |
| Positions | active, liquidatable, closed, zero-debt |
| Tokens | 18-decimal (ETH/WETH), 6-decimal (USDC), 8-decimal (WBTC) |
| Transactions | pending, confirmed, failed, gas-bumped |

## Naming Convention

```
<entity>_<variant>
```

Examples: `position_active`, `position_underwater`, `user_funded`, `token_usdc`.

Use factory functions over raw objects when fixtures need computed fields:

```typescript
function makePosition(overrides?: Partial<Position>): Position {
  return { id: "pos-test", status: "active", healthFactor: 1.85, ...overrides };
}
```
