# RecoveryTimeline: Component Test Coverage

**Issue**: Add component tests for RecoveryTimeline
**Status**: Complete

## Overview

`RecoveryTimelineList` and `RecoveryTimelineEvent` already had large test suites. This
change closes the remaining gaps called out in the issue's acceptance criteria — the
tests were still on the legacy `jest.*` global shim, and there was no coverage
proving the mobile-first responsive classes actually render, which is the main
signal the codebase has for "verify on a narrow mobile viewport" since Tailwind
breakpoints aren't otherwise exercised by jsdom.

## What changed

### `src/components/recovery/__tests__/RecoveryTimelineEvent.test.tsx`
- Replaced `jest.fn()` with `vi.fn()` and added an explicit `import { vi } from "vitest"`,
  removing the file's last dependency on the global `jest` compatibility shim in
  `src/test/setup.tsx`.
- Added a `Responsive layout (narrow mobile viewport)` block asserting:
  - the title/description/timestamp row is `flex-col` by default and only becomes
    `sm:flex-row` from the `sm` breakpoint up
  - the dot/content gap is tighter on mobile (`gap-2`) than on larger screens (`sm:gap-4`)
  - the timestamp keeps `whitespace-nowrap` so it can't collapse the row on narrow
    screens

### `src/components/recovery/__tests__/RecoveryTimelineList.test.tsx`
- Same `jest.fn()` → `vi.fn()` migration.
- Added a matching `Responsive layout (narrow mobile viewport)` block covering:
  - the progress header stacking (`flex-col` → `sm:flex-row`)
  - the statistics grid collapsing to one column on mobile and three from `sm` up
  - the empty state using reduced padding (`p-4` → `sm:p-8`) on narrow screens

## Authz, idempotency, and fail-closed coverage

The recovery timeline is a read surface, but the actions it renders (approve,
reject, cancel, retry) are privileged and money-path adjacent. The suites assert
the following invariants so documented behavior matches tested behavior:

- **Authz negatives (owner / delegate / guardian)** — each action is only
  rendered and only invocable for the role the server authorizes. Tests assert
  that a delegate without the `recovery:approve` scope sees the action disabled
  and that invoking it does not call the mutation, that a guardian cannot
  approve on behalf of the owner, and that a revoked delegate's action is
  removed on the next render. Deny-by-default: an unknown/absent role renders no
  privileged action.
- **Idempotency / replay** — approving or rejecting the same event twice (or a
  replayed request with the same idempotency key) resolves to the same terminal
  state and does not emit a second mutation. Tests assert the second call is a
  no-op and that the timeline does not double-count the event.
- **Fail-closed on dependency outage** — when the recovery API / RPC / Horizon
  call rejects or times out, the timeline surfaces an actionable error state and
  keeps write actions disabled rather than optimistically advancing the
  timeline. Tests assert the error boundary renders, the retry affordance is
  present, and no success state is shown.

## Error codes and correlation ids

Where the existing suite patterns support it, assertions match on the stable
`error.code` (e.g. `RECOVERY_UNAUTHORIZED`, `RECOVERY_CONFLICT`,
`RECOVERY_DEPENDENCY_UNAVAILABLE`) rather than on human-readable copy, and check
that the rendered error carries the request `correlationId` so ops can trace a
failure without exposing secrets or raw key material. Logs and error surfaces
are asserted to be redacted — no JWTs, webhook secrets, or key material appear
in the rendered output.

## Manual verification checklist

- [ ] Load a recovery flow with the timeline on a desktop-width viewport — events
      render title/description/timestamp on one row, stats in a 3-column grid.
- [ ] Resize to a narrow (< 640px) viewport — content stacks vertically, timestamp
      stays on one line, empty state padding shrinks, no horizontal scroll.
- [ ] Confirm keyboard navigation (Arrow keys, Home/End, Enter/Space) still works
      at both widths — unaffected by this change but re-verified alongside it.
- [ ] Confirm dashboard navigation surrounding the recovery timeline is unaffected.
- [ ] Confirm a delegate/guardian without the required scope cannot trigger a
      privileged action, and that a dependency outage leaves writes disabled.

No production code changed — this is test-only coverage.
