---
name: idor-exploitation
description: Deep technical skill for finding and exploiting IDOR/BOLA — the mindset, the tells to watch in traffic, end-to-end exploitation chains (horizontal read/write, reference transformation, parameter pollution, collection leaks, blind IDOR, IDOR-to-ATO), concrete techniques, and bypass methods. This is the attack catalog; pair with idor-triage for the two-account confirmation gate. Use when hunting object-level authorization failures across REST, GraphQL, mobile, and microservices on an authorized target.
---

# IDOR / BOLA — Exploitation

IDOR (Insecure Direct Object Reference) / BOLA (Broken Object Level Authorization) is the server honoring a client-supplied object reference without checking that the reference belongs to the caller. It is the single most common and most impactful API bug class: one swapped ID reads or rewrites another tenant's data.

The whole game: **every object reference is attacker-controlled. If the server doesn't bind `subject × object` at the point of access, you own every object it addresses.**

## Core Insight / Mindset

- Treat every ID — path, query, body, header, GraphQL node, filename — as untrusted until proven bound to your session.
- Authorization is **object-level**, not endpoint-level. `/api/orders/{id}` may require login (so it "feels" protected) yet never check the order is *yours*. Auth ≠ authz.
- You need **two controlled principals** (A attacker, B victim) and a **known oracle** — a field only B should see. Without B's ground truth you cannot tell a leak from your own data; that gap is the #1 false-IDOR.
- Prefer **read** confirmation; never destroy B's data to prove a write works.

## Attack Surface

- **Direct references:** numeric/sequential, UUIDv1 (time-ordered, guessable), hashed, base64, composite (`tenant:resource`), slugs.
- **Locations:** path segments, query params, JSON/body fields, custom headers (`X-Account-Id`, `X-User`), cookies, GraphQL `node(id:)` and relay global IDs, multipart filenames, WebSocket messages.
- **Carriers of indirect refs:** exports, share links, PDF/report generators, webhooks, signed URLs with weak/omitted binding, batch/bulk endpoints.
- **Multi-service:** a downstream service that trusts an ID forwarded by an upstream without re-checking ownership.

## Signals to Watch (the tells)

- **Sequential or decodable IDs** in requests — increment, decode base64/hex, or parse a UUIDv1 timestamp.
- **An object ID in the body or header that the session already implies** (why send `account_id` if you're logged in? → it may be trusted over the session).
- **List/search endpoints that take a caller-supplied scope** (`?org=`, `?user=`) instead of deriving it from the session.
- **GraphQL global node IDs** (base64 `Type:id`) — decode, re-encode with another id.
- **Share/export URLs** with a predictable token or a raw ID and no per-recipient binding.
- **404 vs 403 differential** — a 403 on another ID proves the object exists and is access-controlled; a 200 proves a problem; a generic 404 everywhere may hide a blind IDOR.
- **Numeric IDs in a mobile/legacy API** that the main web app hides behind opaque tokens — the mobile path often skips the check.

## Discovery / Methodology

1. **Collect references** from recon/proxy: every endpoint that addresses an object, and the ID format.
2. **Establish ground truth as B:** pick one object, record an ID and an oracle field only B sees.
3. **Baseline as A:** call the same endpoint on A's own object — the control shape.
4. **Swap the reference** with A's session: request B's ID, change only the identifier.
5. **Confirm on the oracle,** not the status code (see idor-triage).
6. **Expand by class** (read → write → collection → indirect → blind) only as each is confirmed.

## Exploitation Chains

### Chain 1 — Horizontal read (the core case)
1. As B, note object `1007` contains B's email `victim@…`.
2. As A, `GET /api/orders/1007` with A's token.
3. **Confirm:** response contains B's email. Cross-account read proven.

### Chain 2 — Horizontal write / state change
1. As A, `PATCH /api/orders/1007` (B's) with a reversible change, or `POST /api/orders/1007/cancel`.
2. **Confirm (assisted, reversible):** B's object changed, then revert. Write-IDOR is higher severity than read.

### Chain 3 — Reference transformation
1. IDs are base64 `b3JkZXI6MTAwNw==` → `order:1007`.
2. Re-encode `order:1008`; or decode a UUIDv1 to recover node+timestamp and predict neighbors.
3. **Confirm:** transformed ID resolves to another principal's object.

### Chain 4 — Parameter pollution / scope override
1. Send both IDs: `?id=A_own&id=B_victim`, duplicate JSON keys, or `{"id":"A","account_id":"B"}`.
2. **Confirm:** the server honors the attacker-chosen one (last-wins, or trusts `account_id` over session).

### Chain 5 — Collection / list leak
1. `GET /api/messages?user=B` or a search filtering on a supplied scope.
2. **Confirm:** B's objects appear in A's list — mass extraction, often worse than single-object IDOR.

### Chain 6 — Indirect reference (secondary dereference)
1. Trigger an export/share/report that embeds an object ID.
2. Swap the ID inside the generation request, or access another user's generated artifact by predictable path.
3. **Confirm:** the secondary flow dereferences B's object without re-checking the caller.

### Chain 7 — Blind IDOR (no direct read-back)
1. The write returns no body / a generic 200 and you can't read B's object directly.
2. Use a side channel: a notification/email fired to B, a state visible to B on next login, a timing/length differential, or a related endpoint that reflects the change.
3. **Confirm:** the side channel shows B's state changed.

### Chain 8 — IDOR → Account Takeover
1. A profile/settings endpoint is IDOR-able and accepts `email`/`phone`/`password` or a reset token tied to an ID.
2. Change B's recovery email via the IDOR, then trigger password reset.
3. **Confirm (assisted only, your own test B):** reset lands under attacker control. Highest-impact chain — propose, don't fire autonomously.

## Techniques

- **Enumerate minimally:** confirm one cross-account hit, not a full scrape — the bug is proven by one, scraping is extra risk and often out of scope.
- **Decode everything:** base64, hex, JWT-in-URL, relay global IDs, UUID versions.
- **Verb matrix:** if GET is blocked, try HEAD/OPTIONS/PUT/PATCH/DELETE — enforcement often differs per method.
- **Transport matrix:** REST vs GraphQL vs gRPC-web vs mobile — drive the weakest.
- **Wrap/alias:** array vs scalar, nested object, alternate param name (`uid`, `userId`, `user_id`).

## Bypass Methods

- **Method swap:** read denied on GET, allowed on an alternate verb or `X-HTTP-Method-Override`.
- **Content-type switch** (JSON/form/multipart) to reach a handler without the authz filter.
- **Path confusion:** `..;/`, double-encoding, trailing slash/`.json`, matrix params to dodge a gateway rule while the backend still resolves the object.
- **Transport differential:** GraphQL `node()` returns what REST protects.
- **Edge cache:** an identity-scoped response cached and served cross-account.

A bypass counts only when B's oracle field appears for A.

## Validation

- **Confirmed:** A obtained B's oracle field (read) or changed B's state (reversible write), identities clearly distinct, reproduced. See idor-triage for the full gate.
- Never promote on enumerability or status code alone — authorization, not obscurity, is the test.
- **Evidence + redaction:** the swapped request, the one field proving it is B's, A's distinct identity; redact B's real PII, tokens, cookies. Raw traffic to a debug artifact by ID.

## Scope & Safety

- Read-only confirmation by default; writes only assisted, reversible, never delete.
- Two in-scope accounts only; never target a real third party's object to prove the bug.
- Minimal enumeration; no bulk scrape of B's dataset. ATO chains: propose the sequence, don't fire autonomously.
- Respect scope, rate limits, PII rules; stop and flag if authorization is ambiguous.
