---
name: business-logic-exploitation
description: Deep technical skill for finding and exploiting business-logic flaws — the mindset, the tells to watch in traffic, end-to-end exploitation chains (price/discount abuse, double-spend, state-machine skips, refund abuse, quota bypass), concrete manipulation techniques with example payloads, and bypass methods. This is the attack catalog; pair with business-logic-triage for the low-false-positive confirmation gate. Use when hunting logic flaws in checkout, payments, subscriptions, credits, quotas, or any multi-step money/entitlement workflow on an authorized target.
---

# Business Logic — Exploitation

Business logic flaws abuse *intended* functionality to break a domain invariant: pay less than owed, get refunded more than paid, keep a benefit after the payment reverses, exceed a limit, or retain a privilege after a downgrade. No payload fingerprints them — you need a model of what the application is *supposed* to guarantee, then you find the step where it trusts the client or a prior step.

The whole game: **the server must recompute and re-authorize on trusted state at every transition. Every place it doesn't is a bug.**

## Core Insight / Mindset

- Think in **invariants**, not endpoints. "Value is conserved", "a voucher redeems once", "an order ships only after capture". Each is a sentence you try to make false.
- Think in **actors × actions × resources × time**. The same action behaves differently per role, per sequence, per concurrency, per clock edge.
- The vulnerability lives in the **gap between two trusted steps**: validate→capture, reserve→consume, authorize→settle, approve→execute. Mutate the thing in the gap.
- UI is a suggestion. The real contract is the API sequence and the authoritative store (ledger, counter, entitlement table).

## Attack Surface

- **Financial:** pricing, cart totals, discounts/coupons, tax/shipping, payments (auth/capture/void/refund), credits, gift cards, chargebacks.
- **Account lifecycle:** signup, trial, upgrade/downgrade, suspension, deletion, reactivation.
- **Authorization-by-logic:** feature gates, role transitions, approval/review workflows, impersonation.
- **Quotas/limits:** rate/usage caps, inventory, seats, entitlements, backorder.
- **Multi-tenant:** cross-org data or action bleed through tenant-scoped counters and credits.
- **Async:** webhooks, queue workers, sagas/compensations, cron/backfill jobs — the parts that re-run actions and often skip auth.

## Signals to Watch (the tells)

When you read the target's traffic, these are the tells that a logic flaw is near. Flag any you see:

- **Client-computed money in the request body:** `total`, `price`, `amount`, `subtotal`, `tax`, `shipping` sent from the client instead of only an item ID + quantity. → server may trust client math.
- **State carried in the request:** `status`, `orderState`, `stepToken`, `reviewState`, `approvalId`, `paymentIntentId`. → the state machine may be driven by the client.
- **Separate finalize/confirm/capture endpoints** distinct from the action that set up the state. → the gap to mutate.
- **Idempotency-Key header** or `requestId`/`nonce` in the body. → check its scope (per-path? per-principal? cached?).
- **Discount/coupon fields** applied in their own call, especially one that returns an updated total. → stacking / re-apply / scope abuse.
- **Quantity/amount fields that accept arbitrary values**, and selectors for `currency`, `locale`, `timezone`, `taxRegion`. → negative/zero, rounding, ruleset swaps.
- **Hidden/undocumented fields** in responses that are absent from the documented write (role, owner_id, verified, balance). → mass-assignment-adjacent logic jump.
- **Multi-step flows with a GET or webhook that "confirms"** a step. → replay / out-of-order.
- **Role/feature decisions made at the edge/CDN/gateway** rather than the service that mutates state. → differential bypass.
- **Background side effects** (email, fulfillment, ledger write) triggered async. → double-fire via retry/replay.

## Discovery / Methodology

1. **Map the state machine** of each money/entitlement workflow: states, transitions, and the documented precondition of each transition. Derive it from the UI *and* the proxy traffic (hidden/undocumented API calls matter most).
2. **Build the Actor × Action × Resource matrix:** unauth, basic, premium, staff/admin. For each cell, which actions are allowed and what invariant protects them.
3. **Record invariants explicitly:** conservation of value, uniqueness/idempotency, monotonic counters, exclusivity, precondition-before-transition.
4. **Capture a clean baseline** of the intended flow once, including the authoritative-store state (ledger line, counter, entitlement row). This is your control for every abuse test.
5. **Attack the gaps** with the four variance axes below: sequencing, input, time, concurrency.
6. **Confirm on the authoritative store, not the HTTP 200** (see business-logic-triage for the confidence gate).

## Exploitation Chains

Concrete, end-to-end. Each is a sequence, not a single request.

### Chain 1 — Price/total manipulation (client-trusted math)
1. Add item to cart; observe the checkout request carries `price` or `total` in the body.
2. Replay the checkout, lowering `price` (or `total`) while keeping the item ID.
3. Complete payment.
4. **Confirm:** order ledger shows the lowered amount captured. Invariant broken: price integrity.
*Variant:* change `price` *after* a quote/validate step but *before* capture (gap mutation) when a single body isn't trusted but the two-step flow is.

### Chain 2 — Discount stacking / re-apply
1. Apply coupon A (own call, returns updated total).
2. Apply coupon B that is mutually exclusive with A — in parallel, or by replaying the apply call with a different code before the first commits.
3. Or: apply the same single-use coupon twice via replay; or apply it, remove the qualifying item, keep the discount.
4. Checkout.
5. **Confirm:** final captured total reflects both/duplicate discounts. Invariant broken: discount exclusivity / once-per-user.

### Chain 3 — Double-spend via idempotency/race
1. Identify a redeem/transfer/refund endpoint (read-modify-write).
2. Fire N≈5–10 identical requests in a single burst (or reuse the same Idempotency-Key across them).
3. **Confirm:** two redemptions / two refunds / negative balance on the ledger, where the sequential baseline allowed one. Invariant broken: uniqueness. *(See race-conditions-triage for caps.)*

### Chain 4 — Negative / zero quantity → credit generation
1. Find a cart/order/return endpoint accepting `quantity` or `amount`.
2. Submit a negative quantity (or negative refund line) such that the total computes to a credit in your favor, or zero-price a paid item.
3. Checkout / submit return.
4. **Confirm:** account credited or paid item obtained free. Invariant broken: conservation of value.

### Chain 5 — State-machine skip → unpaid fulfillment
1. Map order flow: `created → paid → fulfilled`.
2. Call the `fulfill`/`ship`/`activate` endpoint directly, skipping `paid`, using a known order ID.
3. **Confirm:** order moves to fulfilled without a capture on the ledger. Invariant broken: precondition-before-transition.
*Variant:* reverse order — refund before capture, cancel after ship, verify after finalize.

### Chain 6 — Refund abuse
1. Purchase and consume a benefit (download digital good, use credit).
2. Refund via one channel (UI) and again via another (support/API tool), or submit partial refunds summing above the captured amount.
3. **Confirm:** refunded total > captured total, or refund issued after the benefit was consumed with no post-consumption check. Invariant broken: conservation of value.

### Chain 7 — Privilege/feature retention after downgrade
1. Upgrade to premium; capture a premium-only action and token/claim.
2. Downgrade (or let the trial expire).
3. Replay the premium action; or hit the premium endpoint directly.
4. **Confirm:** premium action still succeeds (stale capability, feature flag checked only client-side, background task using an old entitlement). Invariant broken: entitlement currency.

### Chain 8 — Business logic × IDOR (amplification)
1. A workflow leak exposes another user's order/resource ID (e.g., in a shared-link, export, or webhook).
2. Drive a sensitive transition (refund, cancel, reassign) against that ID from your session.
3. **Confirm:** you altered B's resource. Invariant broken: object-level authorization inside a workflow. *(Pair with idor-triage.)*

## Techniques (how to manipulate)

- **Gap mutation:** alter the value between validate and capture — the single highest-yield move. Swap price, recipient, quantity, or currency after the server "checked" but before it "commits".
- **Replay:** resubmit a stale finalize/confirm/capture; look for duplicate durable effects.
- **Reorder / skip:** call transitions out of documented order via direct API; the server should reject but often doesn't.
- **Limit slicing:** split one over-threshold action into sub-threshold parts to beat a per-transaction cap; confirm the aggregate exceeds the limit.
- **Numeric edges:** negative, zero, `null`, array-instead-of-scalar, scientific notation (`1e3`), very large values, and currency/locale swaps to trigger rounding/truncation in your favor (per-item vs per-order rounding, floating vs decimal).
- **Idempotency-key abuse:** reuse a key across principal/path; or rely on a key stored only in a cache that you can evict/bypass.
- **Channel differential:** the same action via mobile/web/GraphQL/REST often enforces different rules — drive the weakest.
- **Async abuse:** replay a webhook, force a queue retry, or trigger a cron/backfill path that re-runs the effect without re-checking auth or invariants.

## Bypass Methods

- **Content-type switch** (JSON/form/multipart) to reach a handler that skips a check.
- **Method alternation / override** (GET performing a mutation, `X-HTTP-Method-Override`).
- **Client recomputation trust:** totals/tax/discount computed client-side and accepted server-side.
- **Cache/gateway differentials:** stale or non-identity-aware decisions served by CDN/APIM.
- **Internal-header trust:** backend services trusting `X-Role`/`X-User-Id`/`X-Total` injected from an untrusted edge.
- **Partial-failure windows:** two-phase actions where phase 1 commits without phase 2, leaving an exploitable intermediate state.

## Validation

- Prove an **invariant broken on the authoritative source** — two refunds for one charge, negative inventory, quota exceeded, premium after downgrade — not a 200 or an echoed field.
- Show **intended vs abused side-by-side** for the same principal, and demonstrate the undesired state **persists** (ledger, email, admin view).
- **Quantify impact:** unit loss × feasible repetitions, at scale.
- **Minimal evidence + redaction:** the invariant statement, the deciding request(s), the before/after authoritative value; raw traffic to a debug artifact. Redact tokens, cookies, PANs, PII.
- Use the **business-logic-triage** skill for the full confidence-gating rules (confirmed / high / medium / low) and the false-positive list.

## Scope & Safety

- Minimal, reversible, non-destructive PoCs on an authorized, in-scope target. Prefer staging; agents send real payloads that create/modify data.
- Respect rate limits, test windows, and out-of-scope lists (payment flows, mass-email, destructive admin). No persistence, no real exfiltration, no availability impact.
- Writes/money/irreversible actions: default to proposing the exact sequence for operator review rather than firing autonomously. Stop and flag if scope or authorization is ambiguous.
