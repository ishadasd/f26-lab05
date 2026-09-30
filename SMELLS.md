# reservation-service: Smells and One Fix

Reviewed against starter commit `3ad93758e799c43c42a310872a80a6dbcca407a2`.

## Milestone 1: Three smells

### Smell 1 — Phantom complexity in the manager's cache path

- **Classic or agent-specific:** Agent-specific phantom complexity (Lecture 8): missing context about actual writes/free volume can produce this pattern; the original author's reasoning is not observable from source alone.
- **Where:** Original `src/reservationManager.ts`, constructor and `listBookingsForRoom` (lines 2–3, 33, 41, 117–124); `src/cache/queryCache.ts`, `get`/`set`.
- **Principle:** Simplicity/YAGNI: pay for behavior that is actually needed.
- **Cost:** The manager constructs a private empty cache and checks it on every query, but never calls `set` or exposes the cache. Every supported query still reaches storage. Readers must reason about TTL/stale-data behavior that cannot occur here; adding a cache write later without invalidation would introduce stale booking lists.

### Smell 2 — Duplication over reuse in pricing

- **Classic or agent-specific:** Agent-specific duplication over reuse (also classic duplicated knowledge/DRY); missing context is a plausible cause, not a proven account of the generation session.
- **Where:** `src/reservationManager.ts`, `calculatePrice`/`applyDiscounts`; `src/reportGenerator.ts`, `priceOf` and the pricing constants at the top of both files.
- **Principle:** DRY / one authoritative representation of a business rule.
- **Cost:** Changing the premium surcharge, evening discount, long-booking threshold, or rounding order requires synchronized edits. A receipt can disagree with the revenue report if one copy changes alone. Reports currently recalculate using current room data; changing that to historical booked prices is a separate semantic decision.

### Smell 3 — Speculative over-abstraction in notifier construction

- **Classic or agent-specific:** Agent-specific speculative over-abstraction (classic speculative generality), consistent with an underspecified request; this is a diagnosis of structure, not knowledge of the original prompt.
- **Where:** `src/notifications/notifierFactory.ts`, `builders`, `registerChannel`, `registeredChannels`, `createNotificationChannel`; the manager constructor uses only `DEFAULT_NOTIFIER_CONFIG`.
- **Principle:** YAGNI / avoid paying for unneeded variation.
- **Cost:** The only supported `ChannelName` is `email`, yet callers traverse a mutable global registry, factory and config to obtain it. Replacing the default registration can affect later managers, and readers must trace registry initialization to know what sends a confirmation.

## Milestone 2: One small fix

**Which smell:** Smell 1. It has a locally checkable behavioral argument and does not require choosing new business semantics.

**What changed:** Only production file `src/reservationManager.ts`: removed the two cache imports, private cache field, constructor allocation, and impossible cache-hit branch. `listBookingsForRoom` now delegates directly to `storage.findByRoom(roomId)`.

**Scope line:** Remove the manager's inert cache integration only. Keep the standalone `src/cache/` utilities and their exported surface; do not implement caching, alter storage/notifiers/pricing, update dependencies, or change any test. Making caching real would require measured need plus invalidation on create/cancel, which is a different feature. Removing unrelated exported utilities is unnecessary for this fix.

**Behavior evidence:** On 2026-09-30, the untouched starter passed 39/39 tests and `npm run typecheck`; after the fix, the same 39/39 tests and typecheck passed (see `evidence/tests-after.txt` and `evidence/typecheck-after.txt`). `git diff --exit-code 3ad9375 -- tests package-lock.json` confirms no test or lockfile changes. The schedule test exercises empty/populated room lists and summary output; other tests cover booking, cancellation, pricing, reports and validation. The suite is not a proof for every caller. The additional reasoning is that the cache starts empty, is private, and has no population path in this module; consequently the removed branch cannot succeed through the supported API. No performance improvement or external integration test is claimed.

## Milestone 3: Two proposals and one false positive

### Proposal A — Shared pricing rule (not coded)

**Problem:** Smell 2: duplicated pricing policy.

**Decomposition:** Extract one pure `calculateBookingPrice(room, start, end)` function owning surcharge, discounts, and exact rounding order; both the manager and report delegate to it. Keep report aggregation in `ReportGenerator` and booking lifecycle in the manager. Preserve current recalculation semantics unless a separately approved requirement selects historical `booking.priceCents`.

**Cost:** Both callers become coupled to one pricing module, and extraction must preserve rounding/order exactly; future historical-rate reporting needs a separately defined policy/version.

### Proposal B — Explicit notifier dependency (not coded)

**Problem:** Smell 3: a global registry for one supported notifier.

**Decomposition:** Let the manager accept `NotificationChannel` as a constructor dependency, with a direct `EmailChannel` default at the composition boundary. The interface still isolates delivery and supports test doubles; remove registry lookup from manager construction. Audit other users of the exported factory before removing that API.

**Cost:** Construction sites must supply the notifier when varying it, and registry callers would need migration. A genuine requirement for runtime-discovered third-party channels could justify keeping a registry.

### False positive — A validation function with many guards

**Where:** `src/validation.ts`, `validateReservationRequest`.

**Why fine here:** Although it has many `if` statements, they implement one cohesive job: validate a request against its room/building rules and return the first useful error. Shape, time, duration, capacity and opening-hour checks protect different actual constraints. Splitting every guard into a class would obscure the deliberately ordered failures.

**What would change the verdict:** If different buildings introduce independently varying schedules/capacity policies and repeated conditionals appear in multiple validators, isolate those policies while retaining clear validation order.
