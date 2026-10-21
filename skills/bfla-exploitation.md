---
name: bfla-exploitation
description: Deep technical skill for finding and exploiting Broken Function Level Authorization (BFLA, OWASP API5) — the mindset, the tells to watch, end-to-end exploitation chains (vertical privilege escalation, verb/method drift, edge-vs-core mismatch, feature-flag bypass, multi-step re-check gaps, background-job abuse), concrete techniques, and bypass methods. This is the attack catalog; pair with idor-triage/business-logic-triage for the confirmation gate. Use when hunting action-level authorization failures on an authorized target. Distinct from BOLA/IDOR (object-level) — this is about invoking functions a role shouldn't.
---

# Broken Function Level Authorization (BFLA)

BFLA is an *action-level* authorization failure: a caller invokes a function — an endpoint, a mutation, an admin tool — that their role is not entitled to. It is the vertical twin of BOLA/IDOR. BOLA asks "can I reach *another user's object*?"; BFLA asks "can I perform *an action above my privilege*?" (promote myself, refund, delete, impersonate, toggle a flag).

The whole game: **authorization for an action must be enforced at the service that performs it, bound to `subject × action`. Find the privileged function whose check lives only in the UI, the gateway, or the happy-path transport — then call it directly with a lower-privileged principal.**

## Core Insight / Mindset

- **Function-level, not object-level.** The bug is reaching the *operation*, regardless of whose object it touches. Don't confuse with IDOR — here the action itself is the prize.
- **The check is often in the wrong place:** client-side (hidden button), at the edge/gateway, or only on the main transport — while the core service / an alternate transport accepts the call unchecked.
- **You need the role matrix.** At minimum two principals: a low-privileged one (basic user / read-only token) and knowledge of a high-privileged action. The confirmation is: the privileged action *took effect* under the low role.
- **Admin actions are often destructive** (delete, suspend, refund). Prefer reversible/read-shaped privileged actions to prove the flaw; propose destructive ones for operator review rather than firing them.

## Attack Surface

- **Admin/staff/support functions:** user management, impersonation/sudo, org invites, account suspension/reactivation.
- **Money/entitlement actions:** approve/void/refund, credit issuance, price/plan override, seat/license changes, quota grants.
- **Security-sensitive overrides:** 2FA reset, email/phone verification override, role/permission changes, API-key issuance.
- **Operational:** export/report generation, bulk delete, feature-flag toggles, config changes.
- **Transports:** REST, GraphQL mutations, gRPC (reflection), WebSocket, and legacy/mobile API variants of the same action.

## Signals to Watch (the tells)

- **401/403 in the UI but 200 via direct API call** — the clearest tell: enforcement was client-side only.
- **Status differs across transports** for the same action (REST denies, GraphQL/gRPC/mobile allows).
- **Changing only a header** (`X-Role`, `X-User-Id`, `X-Org`, `X-Admin`) alters access without a new token → gateway-injected identity trusted by the backend.
- **Disabled/hidden UI controls** (feature-flagged buttons) whose endpoints are still live — often surfaced in JS (see javascript-recon).
- **Admin-only GraphQL mutations/fields** present in the schema but absent from the UI.
- **A privileged verb on a normal resource** (`DELETE`/`PUT` where you only see `GET`), or `X-HTTP-Method-Override`/`_method` support.
- **Multi-step actions** (create→approve, request→finalize) where step 1 is allowed and a later privileged step isn't re-checked.
- **Background/webhook/job endpoints** that perform privileged work without verifying the caller.

## Discovery / Methodology

1. **Enumerate privileged functions:** admin/staff APIs, GraphQL mutations (introspection), gRPC services (reflection), legacy/mobile endpoints, and endpoints referenced in JS but not reachable from the UI.
2. **Build the Actor × Action matrix:** for each privileged action, which role *should* perform it, and which principals you control.
3. **Baseline as the privileged role** (if you have one, even a test account) to learn the exact request; otherwise reconstruct it from JS/schema/docs.
4. **Replay as the low-privileged principal**, changing only the identity (token) — keep the request otherwise identical.
5. **Confirm the action took effect** under the low role on an authoritative source (not just a 200). Prefer reversible proofs.
6. **Gate** (idor-triage / business-logic-triage) before reporting.

## Exploitation Chains

### Chain 1 — Vertical escalation (basic → admin function)
1. Identify an admin action, e.g. `POST /api/admin/users/{id}/role {"role":"admin"}`.
2. Call it with a basic-user token, otherwise identical.
3. **Confirm:** the role change took effect (read-back / the elevated action now works). Revert.

### Chain 2 — Verb / method drift
1. `GET /api/users/{id}` is allowed; `DELETE`/`PUT` is UI-gated.
2. Send the privileged verb directly, or via `X-HTTP-Method-Override: DELETE` / `_method=DELETE`.
3. **Confirm (reversible-first):** the privileged verb executed under your role.

### Chain 3 — Edge vs core mismatch / header trust
1. The gateway blocks the action, but the backend trusts `X-Role`/`X-User-Id` it normally injects.
2. Supply the header yourself (or hit the internal route directly) with a low-priv token.
3. **Confirm:** backend honors the injected identity over the token claim.

### Chain 4 — Transport differential
1. REST denies `cancelSubscription`/`refund`; the **GraphQL mutation** or **mobile/legacy** endpoint for the same action doesn't check.
2. Invoke via the weaker transport.
3. **Confirm:** privileged action succeeds where the primary path refused.

### Chain 5 — Feature-flag bypass
1. A capability is gated by a client-side flag; the backend endpoint is live regardless.
2. Call the backend directly (URL/mutation found in JS).
3. **Confirm:** the gated function executes without the flag.

### Chain 6 — Multi-step re-check gap
1. A privileged flow is `request → approve/finalize`.
2. You can create the request; call `approve`/`finalize` directly (yours or, chained with IDOR, someone else's).
3. **Confirm:** the privileged finalize executed without the authorization the first step implied.

### Chain 7 — Background-job / webhook abuse
1. A job/webhook endpoint performs a privileged action on trigger.
2. Invoke/replay it directly as a low-priv caller.
3. **Confirm:** the privileged side effect fired without caller authorization.

## Techniques

- **Role-matrix replay:** capture a privileged request, replay verbatim swapping only the token — the cleanest BFLA test.
- **Method tampering:** try all verbs + `X-HTTP-Method-Override`, `_method`, and HEAD/OPTIONS leaks.
- **Header precedence probing:** inject `X-Role`/`X-User-Id`/`X-Org`/`X-Admin`/`X-Forwarded-*` and observe whether backend trusts them over the token.
- **Schema/reflection mining:** GraphQL introspection for admin mutations; gRPC reflection for privileged RPCs; JS for live-but-hidden endpoints.
- **Transport fan-out:** test the same action across REST/GraphQL/gRPC/WebSocket/mobile.

## Bypass Methods

- **Path confusion** to dodge a gateway rule while the backend still routes: `..;/`, double-encoding, trailing `/`/`.json`, matrix params, case variance on `/Admin/`.
- **Content-type switch** (JSON/form/multipart) to hit a handler with weaker middleware.
- **Method override** headers/params where the router honors them.
- **Internal route exposure / SSRF** to reach a core service that assumes it's unreachable.

A bypass counts only when the privileged action *took effect* under the lower role.

## Validation

- **Confirmed:** a privileged function executed under a principal not entitled to it, shown by its effect on an authoritative source (role changed, refund issued, record deleted, flag toggled), reproduced, and reverted where possible. Never promote on a 200 alone — some endpoints return 200 then no-op. Use idor-triage / business-logic-triage for the full confidence gate.
- **Evidence + redaction:** the privileged request, the low-priv identity used, and the authoritative before/after. Redact tokens, cookies, PII. Raw traffic to a debug artifact by ID.
- **False positives:** actions the role is actually entitled to; 403 correctly returned; "success" with no durable privileged effect; admin ops behind proper audit/approval.

## Scope & Safety

- Prefer reversible/read-shaped privileged actions to prove the flaw; **propose destructive admin actions (delete/suspend/refund) for operator review rather than firing them autonomously.**
- Use in-scope test accounts; never suspend/delete/refund a real third party to prove the bug.
- Always revert any state you changed.
- Respect scope, rate limits, and test windows; stop and flag if authorization is ambiguous.
