---
name: javascript-recon
description: Deep technical skill for hunting in client-side JavaScript — collecting and processing .js (including source maps, webpack chunks, and HAR extraction), the patterns to hunt inside it (hidden endpoints, API routes, secrets/keys, parameters, DOM XSS sinks, feature flags, GraphQL ops), end-to-end chains from a JS finding to a confirmed bug, deobfuscation and diffing techniques, and strict noise control so a found endpoint is treated as a lead, not a vulnerability. Use during recon on an authorized target to map attack surface and extract leads from JavaScript.
---

# JavaScript Recon / Hunting

Client-side JavaScript is the most under-tested map of a target. It ships the full list of API routes the app calls, parameter names the UI never shows, feature flags, GraphQL operations, and — too often — live secrets. The server's attack surface is largely described in the JS it hands you for free.

The whole game: **the JS is a map and sometimes a key. Harvest every bundle, extract the leads (endpoints, params, secrets, sinks), then feed each lead into the matching attack skill. A lead is not a finding until it's confirmed.**

## Core Insight / Mindset

- **JS is a lead generator, not a finding.** A hidden endpoint, a param name, a referenced role — each is a hypothesis to confirm with the class skill (idor, business-logic, mass-assignment…), never a report on its own.
- **Coverage beats cleverness.** The win is harvesting *every* bundle — lazy-loaded chunks, old versions, source maps — not deobfuscating one file perfectly. Most teams test the main bundle and miss the chunk that holds the admin route.
- **Diff over time.** New deploys ship new endpoints and flags before the UI exposes them. Re-pull and diff JS to catch surface that isn't live yet.
- **Secrets need validation, not just a regex hit.** A string matching `AKIA…` is a candidate; it's a finding only when it authenticates — and you redact it everywhere.

## Attack Surface (where JS lives)

- **Loaded bundles:** `<script src>`, inline scripts, module imports, dynamically injected scripts.
- **Lazy chunks:** webpack/vite/rollup split chunks pulled on route/interaction — often hold admin/beta code the landing bundle doesn't.
- **Source maps:** `.js.map` (or `//# sourceMappingURL=`) — recover original, commented source incl. internal paths.
- **Historical JS:** Wayback/archive, previous deploy hashes, CDN cache — old endpoints that still work.
- **Non-HTML carriers:** service workers, web workers, WASM glue, mobile app JS bundles, HAR captures (your own traffic export).

## Signals to Watch (what to hunt inside the JS)

- **Endpoints / routes:** `/api/…`, absolute/relative URLs, path templates with `${id}`, `fetch(`/`axios`/`XMLHttpRequest` call sites, route tables, OpenAPI/Swagger refs.
- **Parameters & fields:** keys passed to requests that the UI never surfaces — `isAdmin`, `role`, `debug`, `internal`, `impersonate`, `owner_id`, `tenant_id` (direct mass-assignment leads).
- **Secrets / keys (candidates):** AWS `AKIA…`/`ASIA…`, Google `AIza…`, Stripe `sk_live_`/`rk_live_`, GitHub `ghp_`/`gho_`, Slack `xox[baprs]-`, JWTs, private keys (`-----BEGIN`), Firebase/Supabase configs, basic-auth in URLs, bearer tokens, `apiKey:`/`secret:`/`password:` literals.
- **GraphQL:** inline `query`/`mutation` strings, operation names, persisted-query hashes, schema fragments.
- **Feature flags / gates:** `flags`, `features`, `experiments`, `beta`, `rollout`, role checks done client-side.
- **DOM XSS sinks:** `innerHTML`, `outerHTML`, `document.write`, `eval`, `Function()`, `setTimeout(string)`, `location`/`href` assignment, `dangerouslySetInnerHTML`, `postMessage` handlers, jQuery `html()/append()`.
- **Infra tells:** cloud bucket URLs, internal hostnames, debug/staging endpoints, versions, maps API keys, 3rd-party SaaS IDs.

## Discovery / Methodology

1. **Enumerate all JS** for the target scope: page scripts, every lazy chunk (crawl + trigger routes), source maps, service workers, and historical copies (archive, old hashes).
2. **Extract from traffic:** pull JS from your own HAR capture and filter to the target's domains/subdomains (e.g., with `hargo`) so you work only in-scope.
3. **Normalize:** beautify/prettify minified code; where `.js.map` exists, recover original source; unpack webpack chunks.
4. **Hunt the patterns** above (endpoints, params, secrets, sinks, GraphQL, flags) — tooling in Techniques.
5. **Deduplicate & classify** each lead by the attack class it feeds; drop noise (3rd-party libs, analytics, public CDNs) unless in scope.
6. **Hand off** each lead to the matching skill for confirmation; gate before reporting.

## Exploitation Chains (JS lead → confirmed bug)

### Chain 1 — Hidden endpoint → authz/IDOR
1. A chunk references `/api/v2/admin/users/{id}` not reachable from the UI.
2. Call it with a normal session; swap IDs.
3. **Confirm** with idor-triage (two-account oracle) or BFLA — undocumented ≠ unprotected until tested.

### Chain 2 — Leaked parameter → mass assignment / logic
1. The request builder includes `role` / `owner_id` / `status` the form never sends.
2. Inject one field into a legitimate write.
3. **Confirm** via read-back (mass-assignment-triage).

### Chain 3 — Live secret → direct access
1. A bundle embeds a key (cloud, SaaS, signing).
2. **Validate minimally and non-destructively** (a read-only identity/whoami call), then stop.
3. **Confirm:** the key authenticates. Report with the key redacted; do not exercise write/billable actions.

### Chain 4 — GraphQL operations → schema & authz
1. JS contains operation strings / persisted-query hashes.
2. Replay them, introspect, probe nested authz.
3. **Confirm** with the graphql skill.

### Chain 5 — DOM sink → DOM XSS
1. A sink (`innerHTML`, `location`, `postMessage` handler) consumes a source you control (URL, hash, message, storage).
2. Trace source→sink; build a minimal proof payload.
3. **Confirm:** execution in the DOM from attacker-controlled input (non-destructive marker, e.g., a benign `alert`-equivalent in your own session).

### Chain 6 — Source map → source recovery
1. `.js.map` is reachable (or recoverable).
2. Restore original source + comments + internal paths.
3. **Lead:** internal routes, TODOs, auth logic, more secrets — feed back into chains 1–5. (Source disclosure itself is usually informational.)

### Chain 7 — JS diff → pre-release surface
1. Diff current JS against a prior pull (or archive).
2. New endpoints/flags appear before the feature ships.
3. **Lead:** test the new surface early.

## Techniques

- **Collection:** crawl for all chunks (katana/gau/waybackurls), `hargo` for HAR→JS filtered by domain, trigger routes to force lazy chunks, pull `.map` files.
- **Processing:** beautifier/prettier for minified code; source-map restore (e.g., source-map tooling / unwebpack) to recover originals; AST walk for sink/source tracing.
- **Pattern hunting:** endpoint extractors (LinkFinder/xnLinkFinder style regexes), secret scanners (trufflehog/gitleaks-style rulesets, nuclei JS templates), custom regex sets for the signals above.
- **Deobfuscation:** beautify first; for packed/obfuscated code, rename-and-trace rather than full unpack; focus on call sites to network and sinks, not every line.
- **Diffing:** store hashed snapshots per deploy; diff to surface added routes/flags/strings.

## Noise Control (this is the point)

- A matched endpoint, param, or secret-shaped string is **informational** until confirmed. Do not report "found /api/admin" or "endpoint disclosure" as a vulnerability.
- Drop third-party libraries, analytics, and public CDN code unless explicitly in scope.
- Group leads; one confirmed chain beats a wall of extracted URLs.
- Secrets: confirm with a single read-only call before claiming; never bulk-test keys.

## Validation

- **Confirmed** only via the downstream class skill: the hidden endpoint is authz-broken, the param binds, the secret authenticates, the sink executes. JS extraction alone is a lead, not a finding.
- **Evidence + redaction:** the minimal extracted snippet (file + context), and the confirming request from the class skill. **Redact every secret** (show only a non-reversible fragment), tokens, and PII. Raw bundles/maps to a debug artifact by reference.

## Scope & Safety

- Harvest and analyze only in-scope origins; filter HAR/collected JS to target domains.
- Secrets: validate with one read-only call, then stop — no write, billable, or destructive use; redact everywhere including logs.
- DOM XSS: prove with a benign marker in your own session; no payloads targeting other users.
- Respect rate limits and robots/scope when crawling; stop and flag if authorization is ambiguous.
