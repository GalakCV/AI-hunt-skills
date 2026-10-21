---
name: race-conditions-exploitation
description: Deep technical skill for finding and exploiting race conditions — the mindset, the tells to watch, end-to-end exploitation chains (double-spend, limit overrun, multi-redeem, reservation leak, multi-step reorder, TOCTOU auth), the delivery techniques that actually win races (single-packet attack, HTTP/2 multiplexing, last-byte sync), and bypass methods. This is the attack catalog; pair with race-conditions-triage for the minimal-burst confirmation gate. Use when hunting TOCTOU/double-spend flaws on an authorized target without hammering it.
---

# Race Conditions — Exploitation

A race condition exists when two or more operations interleave inside a window where the application assumed they could not, breaking an invariant the sequential flow preserves: one voucher redeemed twice, one balance spent twice, one seat assigned to many. The bug is in the **check-then-act gap** — the time between reading state and committing the change.

The whole game: **find a read-modify-write with no atomic lock, then collapse the window by firing concurrent requests so they all pass the check before any commits.**

## Core Insight / Mindset

- Treat every read-modify-write and every multi-step workflow as adversarially concurrent.
- The win condition is **sub-millisecond simultaneity** — requests must hit the gap together. Network jitter is the enemy; the delivery technique matters as much as the target.
- The signal is **binary and durable**: the invariant broke on the authoritative store, or it didn't. Duplicate 200s are not proof.
- Keep bursts **minimal and single-shot** — sustained concurrency is a DoS (out of scope), and more volume adds risk, not evidence.

## Attack Surface

- **Single-request RMW:** redeem coupon, apply credit, transfer funds, cast vote, claim reward, reserve stock, use one-time token.
- **Multi-step with finalize:** cart→checkout→capture, request→approve→execute, upload→scan→publish.
- **Limit/quota resets:** daily/monthly caps, rate limits, trial counters, free-tier allowances.
- **Auth/session TOCTOU:** check-permission-then-act, use-token-then-invalidate, 2FA verify windows.

## Signals to Watch (the tells)

- An endpoint that **reads a balance/counter/status and then writes** based on it (redeem, spend, reserve, promote).
- **Idempotency-Key / nonce / requestId** present — check whether the guard is applied *before* or *after* the effect, and whether it's stored durably or only in cache.
- **A value that should be one-time** (coupon, invite, gift card, password-reset token, referral).
- **A quota with a hard cap** and a reset boundary (`resets at 00:00 UTC`).
- **Multi-step flows** where a later step trusts an earlier step's state without re-locking.
- **"Pending → confirmed" money flows** (auth/capture, hold/settle) with a visible gap.

## Discovery / Methodology

1. **Name the invariant:** "voucher redeems at most once", "balance never goes negative", "≤ N seats".
2. **Capture the sequential baseline:** run the action once, serially; record the authoritative result. This is the control the race must break.
3. **Prepare a minimal burst** (N≈5–10) of the identical request, queued for simultaneous release.
4. **Deliver with a sync technique** (see Techniques) to collapse jitter.
5. **Read the authoritative store** — invariant broken? (See race-conditions-triage for caps and the gate.)
6. **One confirming re-run** maximum on write endpoints.

## Exploitation Chains

### Chain 1 — Coupon / gift-card multi-redeem (double-spend)
1. One-time coupon worth $20; sequential redeem works once, second returns "already used".
2. Fire 10 redeems in a single synchronized burst.
3. **Confirm:** ledger shows 2+ redemptions / balance credited multiple times.

### Chain 2 — Balance / funds double-spend
1. Account has $50; transfer/withdraw reads balance then debits.
2. Burst two $50 transfers simultaneously.
3. **Confirm:** both succeed, balance goes negative or two payouts occur. Invariant: conservation of value.

### Chain 3 — Limit / quota overrun
1. "One vote per user", "3 invites max", "1 trial per account".
2. Burst N identical create/claim requests.
3. **Confirm:** aggregate exceeds the cap (4 votes, 10 invites, 2 trials).

### Chain 4 — Reservation / inventory leak
1. Reserve-then-confirm flow with limited stock.
2. Burst reservations; complete one; check the others are not released.
3. **Confirm:** stock oversold or holds leaked (negative/over-allocated inventory).

### Chain 5 — Multi-step reorder under load
1. `authorize → capture` or `verify → finalize`.
2. Concurrently fire finalize before verify completes, or capture twice against one authorization.
3. **Confirm:** an intermediate/duplicate state persisted that the sequential flow forbids.

### Chain 6 — Auth TOCTOU
1. A token/permission is checked then consumed (password-reset, email-verify, one-time 2FA).
2. Burst uses of the same single-use token.
3. **Confirm:** the token is honored more than once (e.g., reset applied twice, or a revoked session still acts in the window).

## Techniques (how to actually win the race)

- **Single-packet attack (HTTP/2):** place the last byte of N requests into one TCP packet so the server receives them simultaneously — the most reliable modern method; defeats network jitter. (Burp Repeater "send group in parallel" / Turbo Intruder `engine=Engine.BURP2`/HTTP2.)
- **Last-byte synchronization (HTTP/1.1):** send all but the final byte of each request, then release the final bytes together over warmed connections.
- **Connection warming:** send a dummy request first to complete TLS/slow-start so the real burst isn't skewed by handshake latency.
- **HTTP/2 multiplexing:** many streams on one connection reduce per-request overhead and tighten timing.
- **Keep N small:** 5–10 is enough to prove the window; ramping is DoS-shaped and adds no evidence.

## Bypass Methods

- **Weakest handler:** the same action via REST vs GraphQL vs mobile may lock differently — race the one without the lock.
- **Async/background path:** a queue retry or webhook that re-runs the effect without an idempotency guard.
- **Multi-node inconsistency:** a counter consistent only per-node — the race needs concurrency across nodes, not a payload.
- **Cache-based guard:** an idempotency/once-check stored only in a cache you can evict or that isn't read atomically.

## Validation

- **Confirmed:** invariant broken under the burst but not under the sequential baseline; minimal burst; one re-run. See race-conditions-triage for the full gate.
- Never promote on duplicate 200s alone — only a broken invariant on the authoritative store counts.
- **Evidence + redaction:** invariant, sequential baseline result, post-burst authoritative result showing the violation; redact tokens/PII; raw burst traffic to a debug artifact by ID.

## Scope & Safety

- Minimal burst (N≈5–10), single endpoint, single attempt — never loop or ramp, never sustain concurrency (that is a DoS).
- Writes/money/irreversible: assisted by default; one confirming re-run maximum.
- Respect rate limits, test windows, scope; stop and flag if authorization is ambiguous.
