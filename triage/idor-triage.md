---
name: idor-triage
description: Operational playbook for triaging and confirming IDOR/BOLA with high signal and low false positives — consumes recon output, needs two controlled accounts, confirms cross-account access on an authoritative oracle, and gates each finding to a confidence level before reporting. Works both recon-fed/assisted and fully autonomous. Load alongside the idor skill: that one is the technique catalog, this one is the execution and confirmation workflow. Use when triaging object references, deciding whether a suspected IDOR is real, or proving cross-account access without destructive writes.
---

# IDOR / BOLA — Triage & Confirmation

An IDOR is not proven by a 200 on someone else's object ID — it is proven when account A reads or changes account B's resource and the access was never bound to the caller. This skill is the discipline between "the ID is enumerable" and "confirmed cross-account access". Default to read-only confirmation; never alter another user's data to prove a point.

Pair with the `idor` skill for the catalog (reference types, chaining, bypass surface). This skill governs *how you run and confirm*.

## Attack Surface

- **Intake from recon first.** Pull every endpoint carrying an object reference: path IDs, query params, body fields, GraphQL node IDs, headers (X-Account-Id, tenant hints). Prioritize references to money, PII, documents, messages, and admin objects.
- **You need two controlled principals.** Account A (attacker) and account B (victim), both yours or in-scope test accounts. Without B's known object ID as ground truth, you cannot distinguish a leak from your own data — that gap is the #1 source of false IDOR reports.
- Discard references with no sensitive object behind them (enum of public catalog IDs is not IDOR).

## Methodology

1. **Establish ground truth.** As B, record an object ID and its exact contents (the oracle: a field only B should see).
2. **Capture A's baseline.** As A, call the same endpoint on A's own object. This is the control shape.
3. **Swap the reference.** As A, request B's object ID. Change *only* the identifier; keep A's session/token.
4. **Confirm on the oracle, not the status.** A 200 is not enough — the response must contain B's known field, or B's state must change (read-only preferred). If it returns A's object or a generic error, it is not IDOR.
5. **Gate and stop** (see Confidence gating).

### Autonomous vs assisted

- **Assisted (recon-fed):** output the A-session request with B's ID substituted, the oracle field to look for, and the expected negative (A's own data / 403). Do not fire traffic.
- **Autonomous:** run the swap read-only, check the oracle, report the gated finding.

Default to **read** confirmation always. Only attempt a write (PUT/PATCH/DELETE on B's object) in assisted mode, with an immediately reversible change, and never a delete.

## Techniques

- **Horizontal:** A reads/edits B's object at the same privilege level — the core case; confirm with B's oracle field.
- **Reference transformation:** sequential/UUID/hashed/base64/composite IDs — decode and increment; confirm the transformed ID resolves to B.
- **Parameter pollution:** send both A's and B's ID (duplicate param, array, nested object); confirm which the server honors.
- **Sibling/collection leak:** list endpoints that filter by caller-supplied scope instead of session; confirm B's objects appear in A's list.
- **Indirect reference via secondary lookup:** an export, share-link, or webhook that dereferences an ID without re-checking the caller.

## Bypass Methods

- Transport differential: REST enforces but GraphQL node query does not; mobile API path skips the check.
- Method swap: GET denied, but HEAD/OPTIONS/alternate verb returns the object or leaks headers.
- Content-type switch to reach a handler without the authz filter.
- Edge/CDN caching of an identity-scoped response served cross-account.

Each only counts when B's oracle field appears for A.

## Validation

### Confidence gating (report nothing above its evidence)

- **Confirmed:** A obtained B's oracle field (read) or changed B's state (reversible write), reproduced ≥2×, with A's and B's identities clearly distinct in the evidence.
- **High:** B's object ID returns 200 with B-shaped data but the oracle field was not pre-recorded — re-run with a known B field to confirm.
- **Medium:** enumerable/guessable references with weak access differences; needs the two-account comparison.
- **Low / informational:** enumerable IDs with proper 403, or access to non-sensitive/public objects. **Not a vulnerability.** Keep out of the main output.

Never promote on enumerability or status code alone — authorization, not obscurity, is the test.

### Evidence & redaction

- Minimal evidence: the swapped request, the oracle field proving it is B's, and A's distinct identity. No full dumps — raw traffic to a debug artifact by ID.
- **Redact** B's real PII, tokens, cookies, and auth headers. Show only the single field that proves cross-account binding failed; mask the rest.

### False positives to drop on sight

- Enumerable IDs returning the caller's own data.
- Public/shared-by-design objects (published posts, public profiles).
- 403/404 on the swapped ID (access correctly denied).
- Objects the caller is legitimately entitled to via role/membership.

## Safety Caps (both modes)

- Read-only confirmation by default; writes only in assisted mode, reversible, never delete.
- Two in-scope accounts only; never target a real third party's object to "prove" the bug.
- Minimal enumeration — confirm one cross-account hit; do not scrape B's dataset or bulk-pull IDs.
- Respect rate limits, scope, and PII-handling rules; stop and flag if scope/authorization is ambiguous.
