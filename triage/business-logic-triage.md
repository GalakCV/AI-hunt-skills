---
name: business-logic-triage
description: Operational playbook for triaging and confirming business-logic flaws with high signal and low false positives — consumes recon output, maps one invariant at a time, runs minimal non-destructive PoCs, and gates each finding to a confidence level before reporting. Works both recon-fed/assisted and fully autonomous. Load alongside the business-logic skill: that one is the technique catalog, this one is the execution and confirmation workflow. Use when triaging business-logic candidates, deciding whether a suspected logic flaw is real, or running abuse tests under rate and concurrency limits.
---

# Business Logic — Triage & Confirmation

A suspected business-logic flaw is worthless until an invariant is shown broken on an authoritative source. This skill is the discipline between "this looks off" and "confirmed": consume recon, pick one invariant, prove it with the smallest reversible PoC, attach a confidence level, and stop. Default to reporting less, not more.

Pair with the `business-logic` skill for the technique catalog (state machines, idempotency, currency, quotas, sagas). This skill governs *how you run and confirm*, not *what classes exist*. If both are loaded, never duplicate the catalog here — act on it.

## Attack Surface

Start from what recon already produced; do not re-crawl what you were handed.

- **Intake from the recon tool first.** Pull finalize/confirm/capture endpoints, state tokens (orderStatus, stepToken, paymentIntentId, approvalId), role-gated actions, and any endpoint touching money, quota, or entitlement. Treat the recon output as the surface; expand only if a confirmed lead points outside it.
- **Prioritize by invariant at stake, not by endpoint count.** One money/entitlement invariant outranks fifty informational endpoints.
- **Discard surfaces with no durable state change** (read-only, cosmetic, visual-only). They cannot host a logic flaw and are pure noise.

Target invariants — pick exactly ONE per test:

- Conservation of value: ledger in == ledger out; no refund exceeds its capture.
- Uniqueness / idempotency: one redemption, one fulfillment per key.
- Monotonicity: counters and quotas never decrease illegitimately.
- Exclusivity: one active subscription / seat / hold per entitlement.
- Authorization-by-logic: a transition requires its documented precondition, enforced server-side.

## Methodology

1. **Select one invariant.** State it as a falsifiable sentence ("a voucher redeems at most once"). If you cannot state it, you cannot confirm a violation — skip the candidate.
2. **Capture the baseline.** Run the intended flow once and record both the request sequence and the authoritative-source state (ledger, quota counter, admin view, confirmation email). This is your control.
3. **Form one hypothesis.** The single sequencing, timing, or input change you believe breaks the invariant.
4. **Run the minimal PoC.** The smallest reversible action that would prove the violation — non-destructive, in scope, within rate limits (see Safety Caps).
5. **Read the authoritative source, not the HTTP 200.** A success response is not confirmation; a changed ledger/counter/entitlement is.
6. **Gate the finding** (see Confidence gating) and stop. Never re-run a confirmed write-PoC to "double-check".

### Autonomous vs assisted (handle both)

- **Assisted (recon-fed):** output the exact request sequence to replay/modify in the proxy, the invariant under test, and the authoritative source to check afterward. Do not fire active traffic yourself.
- **Autonomous ("do it all"):** execute the sequence under the Safety Caps below, read the authoritative source, then report the gated finding.

Decide per task from the user's instruction. When it is unspecified, default to **assisted** for any write, money, or irreversible action, and only go autonomous for read-only probing.

## Techniques

Operational cuts that most often separate a real flaw from noise. Each is "confirmed" only when the invariant breaks on the authoritative source.

- **Late mutation:** alter price / quantity / recipient *between* a validate step and a capture step; confirm the capture used the mutated value on the ledger.
- **Replay of finalize:** resubmit a stale confirm/capture; confirm a duplicate *effect* on the authoritative source, not a duplicate 200.
- **Key-scope abuse:** reuse an idempotency key across a different principal or path; confirm cross-principal effect.
- **Limit slicing:** split one over-threshold action into sub-threshold parts; confirm the aggregate exceeds the documented cap.
- **Concurrency on atomic checks (race):** fire N identical requests in one burst at a single endpoint (create / redeem / transfer / reserve); confirm the invariant broke (e.g., two redemptions). Keep N minimal — see Safety Caps.
- **Boundary values:** 0, negative, null, array-instead-of-scalar, scientific notation, currency/locale swap — only where the field feeds a money or quota computation.

## Bypass Methods

- **Content-type switch** (JSON / form / multipart) to reach a different handler that skips the check.
- **Method override** (X-HTTP-Method-Override, GET performing a mutation) where edge and core disagree.
- **Channel differential:** mobile / web / GraphQL / REST paths enforcing different rules for the same action.
- **Edge/CDN decision caching** that is not identity-aware.
- **Background path:** webhooks, queue retries, cron/backfill jobs that re-run an action without re-checking auth or invariants.

For every bypass: reaching a different handler is not a flaw proven. It is a flaw only when the invariant breaks on the authoritative source.

## Validation

### Confidence gating (report nothing above its evidence)

- **Confirmed:** invariant shown broken on an authoritative source, reproduced ≥2× (or once for an irreversible write), with a minimal PoC and a side-by-side intended-vs-abused comparison for the same principal.
- **High:** strong indirect evidence (e.g., duplicate 200 plus a matching duplicate confirmation email) but the authoritative source not yet read. Report as *suspected*; name the single check that would confirm.
- **Medium:** anomalous behavior consistent with a flaw but plausibly policy-allowed; needs the baseline comparison to resolve.
- **Low / informational:** different handler reached, odd status, missing client-side check — no durable state change observed. **Not a vulnerability.** Keep it out of the main output.

Never promote severity on a status code, response size, technology/version, or a missing header alone. Correlate evidence before raising severity.

### Evidence & redaction

- **Minimal evidence only** in the main finding: the invariant statement, the deciding request(s), and the before/after authoritative value. No full HTTP dumps — raw traffic goes to a debug artifact, referenced by ID.
- **Redact** tokens, cookies, auth headers, PANs, emails, and personal data in every output and log. If a value must appear to prove the flaw (e.g., a ledger amount), show only that value.

### False positives to drop on sight

- Documented promotions, goodwill credits, free trials.
- Visual or cosmetic inconsistency with no durable effect.
- Admin-only operations with proper audit and approval.
- "Success" responses with no authoritative-source change.

## Safety Caps (both modes)

- One invariant, one hypothesis, one minimal PoC per test.
- **Race PoCs:** minimal burst (start ~5–10 parallel), single endpoint, single attempt — do not loop or ramp.
- **Writes / money / irreversible:** assisted mode by default; never re-run a confirmed write-PoC.
- Respect program rate limits, test windows, and out-of-scope lists (payment flows, mass-email, destructive admin, third-party SSO).
- No persistence, no exfiltration of real data, no availability impact.
- If scope or authorization is ambiguous, stop and flag before any active test.
