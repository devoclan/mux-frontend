# End-to-end tests

Playwright-based end-to-end coverage for mux-frontend critical paths (wallet,
account abstraction, payments).

## Running

```bash
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
