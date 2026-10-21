---
name: race-conditions-triage
description: Operational playbook for triaging and confirming race conditions with high signal and low false positives — identifies the read-modify-write or multi-step flow from recon, captures a sequential baseline, fires a minimal single-burst PoC, and confirms a broken invariant on an authoritative source. Works both recon-fed/assisted and fully autonomous, with strict concurrency caps. Load alongside the race-conditions skill: that one is the technique catalog, this one is the execution and confirmation workflow. Use when triaging TOCTOU/double-spend candidates or deciding whether a suspected race is real without hammering the target.
---

# Race Conditions — Triage & Confirmation

A race is proven only when concurrent requests break an invariant that the sequential flow preserves — two redemptions, a negative balance, a quota exceeded. Duplicate 200s are not proof; a changed authoritative source is. This skill keeps the burst minimal and the confirmation rigorous, so you neither miss the bug nor DoS the target.

Pair with the `race-conditions` skill for the catalog (RMW classes, special contexts, chaining). This skill governs *how you run and confirm*.

## Attack Surface

- **Intake from recon first.** Pull single-request read-modify-write endpoints (redeem, apply, transfer, reserve, vote, claim) and multi-step flows with a finalize/capture. These are the only places a race lives.
- Prioritize by invariant at stake: value conservation, one-time uniqueness, non-decreasing counters, exclusivity.
- Discard idempotent/read-only endpoints — they cannot race into a durable violation.

## Methodology

1. **Name the invariant.** "This voucher redeems at most once." If you cannot state it, you cannot confirm a violation.
2. **Capture the sequential baseline.** Run the action once, serially; record the authoritative result (ledger, counter, balance). This is the control the race must break.
3. **Fire a minimal burst.** Send N identical requests as close to simultaneous as the proxy allows (single-packet / parallel send), one endpoint, one attempt. Start N ≈ 5–10.
4. **Read the authoritative source.** Confirm the invariant broke (e.g., 2 redemptions for a one-time voucher). Duplicate 200s without a durable double-effect are **not** confirmation.
5. **Gate and stop.** Do not escalate N or loop to "get a cleaner repro" on a write endpoint.

### Autonomous vs assisted

- **Assisted (recon-fed):** output the exact request to parallelize, the invariant, the baseline to capture first, and the authoritative source to re-read. Do not fire the burst.
- **Autonomous:** capture baseline, fire the minimal burst, read the source, report the gated finding.

Writes/money/irreversible: assisted by default. Autonomous only for read-only timing probes.

## Techniques

- **Single-endpoint RMW burst:** parallel identical requests at the RMW action; the classic double-spend.
- **Multi-step reorder under load:** fire finalize before verify, or refund before capture, concurrently; confirm an intermediate state persisted.
- **Limit-window edges:** burst at T−1s/T+1s around a quota reset (mind UTC vs local); confirm double-consumption across the boundary.
- **Reservation leak:** reserve in parallel, complete one, confirm the others were not released.
- **Idempotency-key race:** same key, concurrent sends; confirm whether the guard is applied before or after the effect.

Keep every burst minimal and single-shot — the signal is binary (invariant broke or not), so more volume adds risk without adding evidence.

## Bypass Methods

- Different handlers for the same action (REST vs GraphQL vs mobile) with inconsistent locking — race the weakest.
- Background/async path (queue retry, webhook) that re-runs the effect without an idempotency guard.
- Multi-node deployments where a counter is only locally consistent — the race needs concurrency, not a payload.

## Validation

### Confidence gating (report nothing above its evidence)

- **Confirmed:** invariant broken on the authoritative source under the burst, not under the sequential baseline; reproduced with a minimal burst (one re-run is enough — do not spam writes).
- **High:** duplicate side effects observed (two emails, two ledger lines) but authoritative balance/counter not yet read — name the one check to confirm.
- **Medium:** timing anomaly or occasional duplicate response with no confirmed durable effect; needs the baseline-vs-burst comparison.
- **Low / informational:** duplicate 200s with identical idempotent outcome, or no durable change. **Not a vulnerability.**

Never promote on response duplication alone — only a broken invariant counts.

### Evidence & redaction

- Minimal evidence: the invariant, the sequential baseline result, the post-burst authoritative result showing the violation. Raw burst traffic → debug artifact by ID.
- **Redact** tokens, cookies, auth headers, PANs, PII. Show only the deciding authoritative value (e.g., "balance = −50" or "redemptions = 2").

### False positives to drop on sight

- Duplicate responses with a single durable effect (guard worked, response did not dedupe).
- Retries/duplicates explicitly tolerated by design with later reconciliation.
- Timing variance with no invariant violation.

## Safety Caps (both modes)

- Minimal burst: start N ≈ 5–10, single endpoint, single attempt — never loop, never ramp to "amplify" a confirmed bug.
- Writes/money/irreversible: assisted by default; one confirming re-run maximum.
- No sustained concurrency against the target (that is a DoS, which is out of scope and destructive).
- Respect rate limits, test windows, scope; stop and flag if scope/authorization is ambiguous.
