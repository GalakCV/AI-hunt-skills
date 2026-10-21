---
name: graphql-exploitation
description: Deep technical skill for finding and exploiting GraphQL APIs — endpoint discovery, introspection and schema recovery, the tells to watch, end-to-end exploitation chains (authz/IDOR at depth, injection through resolvers, batching brute-force and OTP/2FA bypass, DoS by depth/duplication, SSRF, file-upload, subscription/WebSocket bypass), concrete query payloads, and WAF/introspection/rate-limit bypass. This is the attack catalog; pair with the relevant triage skill for the confirmation gate. Use when a target exposes a GraphQL endpoint on an authorized engagement.
---

# GraphQL — Exploitation

GraphQL moves authorization, input validation, and rate limiting from a fixed set of REST routes into resolvers the client drives. One endpoint, one schema, arbitrary field selection — so the bugs cluster in a few places: the server checks auth at the top of a query but not on nested fields, validates a value's *type* but not its *content*, and limits *HTTP requests* while the client sends many operations in one.

The whole game: **the schema is the attack surface. Recover it, then hit the resolver that validates type but not content, authorizes the parent but not the child, or counts requests but not operations.**

## Core Insight / Mindset

- **The schema is the map.** Introspection (or its recovery when disabled) hands you every query, mutation, type, and argument. Get it first.
- **Authorization is per-resolver.** Top-level `user(id)` may be checked while `user.orders.paymentDetails` deeper down is not — authz is often enforced only at the entry field.
- **Type ≠ safety.** A field typed `String` still carries `' OR 1=1--`. Resolvers that concatenate arguments into SQL/NoSQL/OS commands are injectable exactly like REST.
- **One request, many operations.** Aliases and array batching collapse N operations into one HTTP request — the basis for rate-limit/OTP/brute-force bypass.

## Attack Surface

- **Endpoints:** `/graphql`, `/api/graphql`, `/graphiql`, `/v1/graphql`, `/query`, `/gql`, Hasura/Apollo/Relay defaults; plus the subscription WebSocket (`/graphql`, `/subscriptions`).
- **Operations:** queries, mutations, subscriptions; introspection (`__schema`, `__type`); batching (array and alias).
- **Resolvers** feeding SQL/NoSQL/ORM/OS/HTTP sinks via arguments and filter/orderBy inputs.
- **File-upload** multipart mutations; **URL/webhook** fields (SSRF).

## Signals to Watch (the tells)

- A **JSON POST with `{"query": "..."}`** or a `query`/`operationName`/`variables` shape → GraphQL.
- **`__schema`/`__type` accepted** → full introspection; dump it.
- **Field-suggestion errors** ("Did you mean `email`?") → schema partially recoverable even with introspection off.
- **Arguments that look server-concatenated:** `filter`, `search`, `orderBy`, `name_contains`, raw `id` strings → injection candidates.
- **Deeply nested relations** in the schema (User→orders→paymentDetails; User→friends→User) → nested-authz and recursive-DoS candidates.
- **`login`/`verifyOTP`/`resetPassword` mutations** → batching brute-force / rate-limit bypass.
- **Upload or `url`/`importUrl`/`webhook` fields** → SSRF / path traversal.
- **A subscriptions WebSocket** separate from the HTTP endpoint → may skip the HTTP middleware that enforces auth/introspection/complexity.

## Discovery / Methodology

1. **Find the endpoint** (paths above; `graphw00f` for server fingerprint).
2. **Introspect** (`__schema` full dump; `__type(name:)` if `__schema` is blocked). If both blocked, **recover** the schema from field-suggestion errors (`clairvoyance`) or known types.
3. **Map the schema:** list queries, mutations, subscriptions, argument types, and nested relations. Note which operations touch money/PII/admin and which take free-form string args.
4. **Pick targets** by class (authz-at-depth, injection, batching, DoS, SSRF, upload) and run one hypothesis at a time.
5. **Confirm** on an authoritative source / oracle (see the matching triage skill); gate before reporting.

## Exploitation Chains

### Chain 1 — Nested / field-level authz (IDOR at depth)
1. Introspect to find a type reachable from an object you *can* query.
2. Request sensitive child fields the top-level check doesn't cover:
   `query { user(id:"1"){ orders{ paymentDetails{ cardNumber cvv } } } }`
3. Swap `id`, or batch IDs with aliases: `{ u1:user(id:"1"){email} u2:user(id:"2"){email} }`.
4. **Confirm:** another principal's data returns (use the two-account oracle from idor-triage). *Tell for missing scope:* a read-only token that gets "not found" instead of "scope insufficient" means the resolver ran.

### Chain 2 — Injection through a resolver
1. In a string argument / filter / `orderBy`, send a probe:
   - SQLi: `user(name:"admin' OR '1'='1")`, boolean/union/`orderBy` concat, time-based `id:"1' AND (SELECT pg_sleep(5))--"`.
   - NoSQL (Mongo): operator injection `{ user(filter:{ password:{ "$gt":"" } }) }`, regex extraction, `$where` JS if enabled.
   - OS command: args feeding a report/export shell call.
2. **Confirm:** boolean oracle flips, time-delay reproduces, or union/error leaks data. Time-based first where blind.

### Chain 3 — Batching brute-force / OTP-2FA bypass
1. Target `login`/`verifyOTP`.
2. Send many attempts in **one** request via aliases or an array batch:
   `mutation { v0:verifyOTP(code:"000000"){success token} v1:verifyOTP(code:"000001"){success token} … }`
3. **Confirm:** one alias returns success/token — rate limiting counted requests, not operations. (Keep PoC bounded; prove the bypass on a small range, don't run the full 10^6 — see Safety.)

### Chain 4 — Rate-limit / anti-automation bypass
1. Wrap a limited action (`resetPassword`, invite, vote) in aliases/batch, or vary `operationName`.
2. **Confirm:** N effects from 1 HTTP request where the limit expected N requests.

### Chain 5 — Denial of service (prove, don't weaponize)
1. **Depth/recursion** via circular relations (User→friends→friends…), **field duplication** (same field ×hundreds), **batch** or **directive overload**.
2. **Confirm safely:** a *single, modest* nested/duplicated query showing disproportionate latency or an absent depth/complexity limit — enough to prove the control is missing. Do not sustain load.

### Chain 6 — SSRF via GraphQL
1. A `url`/`importUrl`/`webhook`/upload-by-URL field is server-fetched.
2. Point it at an internal/metadata target (cloud metadata, internal service, `file://`), e.g. the Craft CMS asset-upload `_file` param pattern (CVE-2025-68437).
3. **Confirm:** internal response/metadata returned or a callback to your collaborator host.

### Chain 7 — File-upload abuse
1. Multipart upload mutation.
2. Path traversal in filename, double-extension, null-byte (older stacks), or write to a served directory.
3. **Confirm:** file lands outside intended path / is retrievable. (Non-destructive marker file; no web shell on a live third party.)

### Chain 8 — Subscription / WebSocket bypass
1. The subscriptions WebSocket may skip the HTTP middleware chain (auth, introspection control, complexity) — cf. Parse Server CVE-2026-32594.
2. Connect to the WS endpoint and run operations / introspection without a key, or send complex queries that dodge HTTP-layer limits.
3. **Confirm:** operations succeed over WS that the HTTP endpoint rejects.

## Techniques

- **Introspection recovery when blocked:** `__type` instead of `__schema`; GET/alternate verbs; field-suggestion mining; `clairvoyance`.
- **Aliasing** to run/obfuscate many operations and hide field names.
- **Fragments** to restructure/obfuscate selections.
- **Variables** to separate payloads from the query string (helps past naive filters).
- **Tooling:** `graphw00f` (fingerprint), `InQL`/GraphQL Raider (Burp), `graphql-cop`/`batchql`/`CrackQL`, `clairvoyance`, `graphql-path-enum` (paths to sensitive types).

## Bypass Methods

- **Introspection block:** `__type` vs `__schema`; GET/`?query=`; alternate HTTP methods; a permissive header some builds honor.
- **WAF:** switch `Content-Type` (`application/graphql`, `text/plain`, form); GET with URL-encoded or double-encoded query; whitespace/newlines and `#` comments to break signatures; fragments/aliases to hide field names; benign decoy params.
- **Rate limit:** alias/array batching and varied `operationName` to pack many operations per request.
- **Auth:** header-trust bugs (e.g., Erxes `User` header, CVE-2024-57190); WS path skipping middleware.

## Validation

- **Confirmed** only when the resolver effect is shown on an authoritative source: another principal's data returned, injection oracle reproduced (boolean flip / time delay / leaked row), bounded batch returning a success token, internal/metadata response for SSRF, or an operation succeeding over WS that HTTP rejects.
- Introspection being enabled, or a schema dump, is **informational** — not a vulnerability by itself.
- **Evidence + redaction:** the minimal query, the deciding response field, and the distinct identity/oracle. Redact tokens, PANs, PII, and real victim data; raw traffic to a debug artifact by ID. Use the matching triage skill (idor/business-logic/etc.) for the confidence gate.

## Scope & Safety

- Authorized, in-scope targets only; prefer staging for injection/upload/SSRF.
- **DoS: prove the missing control with a single modest query — never sustain load.**
- **OTP/brute-force: demonstrate the batching bypass on a bounded range, not a full keyspace, against your own test account.**
- SSRF/upload: non-destructive markers; no web shells, no persistence, no real exfiltration.
- Respect rate limits, test windows, and out-of-scope lists; stop and flag if authorization is ambiguous.

---

*Techniques condensed and restructured from the community GraphQL cheatsheet at github.com/aw-junaid/bug-bounty (resources/cheatsheets/GraphQL.md), with CVE references as cited there.*
