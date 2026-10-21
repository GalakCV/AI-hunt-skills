---
name: mass-assignment-triage
description: Operational playbook for triaging and confirming mass assignment with high signal and low false positives — derives candidate fields from recon/response bodies, injects one privileged field at a time, and confirms via read-back that the server actually bound it. Works both recon-fed/assisted and fully autonomous, with reversible changes only. Load alongside the mass-assignment skill: that one is the technique catalog, this one is the execution and confirmation workflow. Use when triaging field-binding candidates or deciding whether a suspected mass-assignment is real without leaving escalated state behind.
---

# Mass Assignment — Triage & Confirmation

Mass assignment is proven only when a client-supplied field the API never documented is actually bound to the model and takes effect — confirmed by reading it back. An accepted request (200, field echoed) is not proof; the field may be silently ignored. This skill injects one field at a time, confirms persistence, and reverts.

Pair with the `mass-assignment` skill for the catalog (input shapes, GraphQL, bypass surface). This skill governs *how you run and confirm*.

## Attack Surface

- **Intake from recon first.** The best candidate fields come from the API itself: fields a GET/response returns but the documented write does not accept (role, is_admin, owner_id, tenant_id, verified, balance, status, price, plan). Diff read-model vs write-model to build the candidate list.
- Also mine: model names in errors, GraphQL introspection input types, mobile app bodies, and sibling endpoints that expose richer fields.
- Prioritize fields that change privilege, ownership, tenancy, or money.

## Methodology

1. **Build the candidate list** from the read/write diff — one privileged field per test.
2. **Baseline:** perform the legitimate update (e.g., change your display name); record the object state afterward. Confirm the privileged field's *current* value as control.
3. **Inject ONE field.** Add the candidate (e.g., `"role":"admin"`) to the otherwise-normal request. Try nested/aliased shapes if flat is rejected (`user[role]`, `role.value`, GraphQL input field).
4. **Confirm via read-back.** Re-fetch the object (or hit a role-gated action). The flaw is real only if the privileged field *changed and took effect* — not merely echoed in the write response.
5. **Revert and gate.** Immediately set the field back to its original value; report at the confirmed level.

### Autonomous vs assisted

- **Assisted (recon-fed):** output the baseline request plus the injected field, the read-back endpoint, and the control value. Do not fire the write.
- **Autonomous:** run the injection with a reversible value, read back, revert, report.

Privilege/ownership/money fields: assisted by default. Autonomous only when the change is trivially reversible and self-scoped (your own object).

## Techniques

- **Flat injection:** add the field at top level of an accepted body.
- **Nested / aliased:** `user[is_admin]=true`, `{"user":{"role":"admin"}}`, dotted keys, GraphQL input object fields the mutation does not document.
- **Type/shape variance:** array vs scalar, string "true" vs boolean, object where a scalar is expected — to slip past a shallow allowlist.
- **Ownership pivot:** inject `owner_id`/`tenant_id`/`account_id` to reassign an object to yourself or across tenants.
- **Status/state jump:** inject `status`/`verified`/`approved` to skip a workflow gate (overlaps business logic — confirm the same way: read-back).

## Bypass Methods

- Content-type switch (JSON/form/multipart) to reach a binder without the allowlist.
- Alternate endpoint/transport (GraphQL mutation, legacy/mobile route) binding more fields than the main REST path.
- Bulk/import endpoints that deserialize whole objects.
- PATCH vs PUT differences where one replaces and over-binds.

Every path counts only when read-back shows the field took effect.

## Validation

### Confidence gating (report nothing above its evidence)

- **Confirmed:** injected privileged field changed the object and took effect, verified by independent read-back (or by the elevated action now succeeding), reproduced ≥2×, then reverted.
- **High:** write response echoes the injected field but read-back not yet done — re-fetch to confirm it persisted vs was ignored.
- **Medium:** field accepted without error but no privileged effect observed; needs the role-gated read-back.
- **Low / informational:** field silently ignored, rejected, or echoed without effect. **Not a vulnerability.**

Never promote on an echoed field or a 200 — binding ≠ acceptance.

### Evidence & redaction

- Minimal evidence: the injected request (one field), the read-back showing the field changed from control to injected value, and the revert. Raw traffic → debug artifact by ID.
- **Redact** tokens, cookies, auth headers, and any real PII that appears in the object; show only the privileged field's before/after.

### False positives to drop on sight

- Fields echoed in the response but not persisted (ignored by the binder).
- Writable-by-design fields the user is entitled to change.
- Validation errors / 403 on the injected field (allowlist held).
- Cosmetic fields with no privilege, ownership, or money effect.

## Safety Caps (both modes)

- One field per test; reversible values only; **always revert** after confirmation.
- No ownership/tenant reassignment of a real third party's object — self-scoped or in-scope test objects only.
- Privilege/money fields: assisted by default.
- Respect scope, rate limits, and test windows; stop and flag if scope/authorization is ambiguous.
