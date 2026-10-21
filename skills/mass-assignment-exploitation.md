---
name: mass-assignment-exploitation
description: Deep technical skill for finding and exploiting mass assignment / auto-binding — the mindset, the tells to watch, end-to-end exploitation chains (privilege escalation, ownership takeover, tenant pivot, workflow-state jump, balance/price injection, GraphQL/nested binding), concrete techniques, and bypass methods. This is the attack catalog; pair with mass-assignment-triage for the read-back confirmation gate. Use when hunting unsafe field binding in REST/GraphQL APIs on an authorized target, with reversible changes only.
---

# Mass Assignment — Exploitation

Mass assignment (auto-binding, over-posting) is a framework binding client-supplied fields straight into a model/DTO without a field-level allowlist. You add a field the API never documented — `role`, `is_admin`, `owner_id`, `balance`, `verified`, `status` — and the server persists it. It is a top driver of privilege escalation and ownership/tenant takeover in modern APIs.

The whole game: **the write model and the authorization model disagree. Find a field the server reads/stores but never meant you to set, inject it in an otherwise-legitimate request, and confirm it bound.**

## Core Insight / Mindset

- The bug is **binding ≠ acceptance**. A 200 or an echoed field proves nothing — the field may be silently dropped. Only an independent **read-back** (or the elevated action now working) confirms it persisted and took effect.
- The best candidate fields come from the **API itself**: anything a GET/response exposes that the documented write doesn't accept.
- Always inject **one field at a time** with a **reversible** value, confirm, then **revert**. Never leave escalated state behind.

## Attack Surface

- REST/JSON, form-encoded, multipart bodies; GraphQL mutation input objects.
- Create and update endpoints (POST/PUT/PATCH), bulk/import endpoints, profile/settings, registration, order/cart, admin DTOs.
- Frameworks prone to it: Rails (`permit`), Spring (`@ModelAttribute`), Laravel (`$fillable`/`$guarded`), Django (`fields = '__all__'`), Express/Mongoose (spread `...req.body`), ASP.NET model binding, Sequelize/TypeORM `create(body)`.

## Signals to Watch (the tells)

- **Fields in a GET response that are absent from the documented write body** (role, owner_id, tenant_id, verified, balance, status, plan, createdBy) → prime candidates to inject.
- **An object round-trips** — you fetch it, edit a couple of fields, PUT the whole thing back → the binder may accept every field you include.
- **Model/entity names leaking** in errors, serializer output, or GraphQL introspection input types.
- **GraphQL `input` types with more fields than the UI uses.**
- **PATCH vs PUT** where PUT replaces the full object (more fields bindable).
- **Nested objects** in the body (`{"user":{...}}`) → try injecting privileged keys inside the nesting.

## Discovery / Methodology

1. **Build the candidate list** by diffing the **read model vs the write model** — fields exposed on GET but undocumented on write. Add introspection/error-leaked fields.
2. **Baseline:** perform the legitimate update (e.g., change display name); record the object state and the *current* value of the target privileged field (control).
3. **Inject ONE field** into the otherwise-normal request.
4. **Confirm via read-back** — re-fetch the object or hit a role-gated action; the field must have *changed and taken effect*, not merely echoed.
5. **Revert** and gate (see mass-assignment-triage).

## Exploitation Chains

### Chain 1 — Privilege escalation (role / is_admin)
1. `PATCH /api/users/me` with `{"name":"x","role":"admin"}` (or `is_admin:true`, `isStaff:true`).
2. Read back `/api/users/me` or try an admin-only endpoint.
3. **Confirm:** role changed and the admin action now succeeds. Revert.

### Chain 2 — Ownership takeover (owner_id / createdBy)
1. On a create/update, inject `{"owner_id":"<me>"}` on an object you shouldn't own, or set it on creation to claim a resource.
2. **Confirm:** object now owned by attacker (read-back shows owner). Pairs with IDOR to reach others' objects.

### Chain 3 — Tenant pivot (tenant_id / org_id)
1. Inject `{"tenant_id":"<other-org>"}` (or your own, to pull an object into your tenant).
2. **Confirm:** object reassigned across the tenant boundary — cross-tenant data movement.

### Chain 4 — Workflow-state jump (status / verified / approved)
1. Inject `{"status":"approved"}`, `{"verified":true}`, `{"kyc":"passed"}` on a flow that is supposed to gate these behind review/payment.
2. **Confirm:** the object skipped its gate (read-back / downstream effect). Overlaps business logic — confirm the same way.

### Chain 5 — Money injection (balance / price / credit)
1. Inject `{"balance":100000}`, `{"price":0}`, `{"creditCents":...}` where a binder over-trusts.
2. **Confirm:** authoritative balance/price changed. High severity; assisted + reversible only.

### Chain 6 — Nested / GraphQL / aliased binding
1. Flat `role` rejected → try `{"user":{"role":"admin"}}`, dotted `user.role`, array shapes, or a GraphQL mutation `input` field the UI never sends.
2. **Confirm:** the nested/aliased path bound where the flat one didn't.

## Techniques

- **Read/write diff** is the highest-yield discovery step — the GET tells you the schema the write forgot to lock.
- **Shape variance:** scalar vs array, `"true"` vs `true`, object vs scalar, to slip past a shallow allowlist.
- **Introspection (GraphQL):** enumerate input object fields; many mutations accept more than the client sends.
- **Enumerate candidate names:** `role, is_admin, isAdmin, admin, owner_id, user_id, account_id, tenant_id, org_id, verified, status, approved, balance, credit, price, plan, permissions, scopes, createdBy`.

## Bypass Methods

- **Content-type switch** (JSON/form/multipart) to reach a binder without the allowlist.
- **Alternate endpoint/transport:** GraphQL mutation or legacy/mobile route binding more fields than the main REST path.
- **Bulk/import endpoints** that deserialize whole objects.
- **PUT over PATCH** to over-bind via full replacement.

Each counts only when read-back shows the field took effect.

## Validation

- **Confirmed:** injected privileged field changed the object and took effect, verified by independent read-back (or the elevated action succeeding), reproduced, then reverted. See mass-assignment-triage for the full gate.
- Never promote on an echoed field or a 200 — binding ≠ acceptance.
- **Evidence + redaction:** the one-field injection, the read-back showing control→injected value, the revert; redact tokens/PII. Raw traffic to a debug artifact by ID.

## Scope & Safety

- One field per test; reversible values only; **always revert**.
- No ownership/tenant reassignment of a real third party's object — self-scoped or in-scope test objects only.
- Privilege/money fields: assisted by default.
- Respect scope, rate limits, test windows; stop and flag if authorization is ambiguous.
