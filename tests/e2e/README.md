# End-to-end tests

Playwright-based end-to-end coverage for mux-frontend critical paths (wallet,
account abstraction, payments).

## Smoke vs full projects

The suite is split into two Playwright projects so contributors and CI can run a
fast critical-path check without waiting on the entire e2e suite:

- **`smoke`** — a curated subset of critical-path specs. This is the fast,
  required check that must stay green on every PR.
- **`full`** — the entire `tests/e2e` suite. This runs as a separate job
  (scheduled / non-blocking) so long-running coverage never blocks the fast path.

### Specs in the `smoke` project

- `tests/e2e/login.spec.ts`
- `tests/e2e/wallets.spec.ts`
- `tests/e2e/wallet-send-receive.spec.ts`

All other specs under `tests/e2e/` belong to the `full` project only. Specs are
not duplicated or stubbed — the same files run in both projects when selected.

### Running locally

```bash
# Fast critical-path subset (required check)
npx playwright test --project=smoke

# Entire e2e suite
npx playwright test --project=full

# Both projects (default)
npx playwright test
```

## Error boundary support correlation

Render/runtime errors are captured by the app-shell error boundary. Each captured
error is assigned a **stable error code** (e.g. `MUX-EB-<category>`) and a
**correlation id** so support can tie a user-visible failure back to structured
logs.

When a test hits the fail-closed fallback UI, assert on the correlation surface
rather than a blank screen:

- The fallback renders a support reference containing the correlation id.
- The same correlation id is emitted in the structured error log (redacted — no
  secrets, JWTs, webhook secrets, or raw key material).

Example assertion pattern:

```ts
await expect(page.getByTestId('error-boundary-fallback')).toBeVisible();
await expect(page.getByTestId('error-boundary-correlation-id')).toHaveText(
  /^[0-9a-f-]{36}$/,
);
```

### Coverage expectations

- Wallet send/receive paths are wrapped by the boundary at the route level.
- AA and payment flows surface the same fallback + correlation id on failure.
- Failures are fail-closed: no partial money-path action proceeds after a
  captured error.

See `docs/security-ux-guards.md` for the security/UX guardrails this boundary
enforces.
