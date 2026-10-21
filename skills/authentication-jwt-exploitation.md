---
name: authentication-jwt-exploitation
description: Deep technical skill for finding and exploiting JWT/OIDC flaws — the mindset, the tells to watch, end-to-end exploitation chains (alg confusion, kid/jku/x5u abuse, signature strip, claim tampering, cross-service aud reuse, weak-secret forgery, token-type confusion), concrete techniques, and bypass methods. This is the attack catalog; pair with auth-jwt-triage for the acceptance-oracle confirmation gate. Use when hunting token-handling flaws on an authorized target, with strict secret redaction.
---

# JWT / OIDC — Exploitation

A JWT flaw is the verifier accepting a token it should reject — forged, tampered, expired, re-keyed, or meant for a different audience. Because JWTs are usually self-contained bearer tokens, a single verification gap often yields durable account takeover or privilege escalation.

The whole game: **the signature and the claims (iss, aud, exp, key id) must all be verified against trusted material bound to this service. Find the one check the verifier skips.**

## Core Insight / Mindset

- The token is not the vulnerability — the **verifier's acceptance behavior** is. A decoded, readable token is expected (JWS is signed, not encrypted).
- Confirmation requires a **protected oracle**: one endpoint that returns 401/403 without a valid token and 200 with one. The flaw is real only when a token-that-should-be-rejected yields 200 or an elevated claim takes effect.
- **Redaction is mandatory.** You will handle live tokens and keys — never output a full valid token, cookie, or private key; mask payload/signature.
- No credential stuffing / brute force / password spraying — those are scope/usage-policy issues, not JWT validation.

## Attack Surface

- JWS (signed) and JWE (encrypted) access/ID/refresh tokens in `Authorization: Bearer`, cookies, or body.
- Header fields that steer verification: `alg`, `kid`, `jku`, `x5u`, `x5c`, `jwk`, `typ`, `cty`.
- OIDC/OAuth2: ID token vs access token, `iss`/`aud`/`azp`/`nonce`, JWKS endpoint, discovery document, multi-issuer trust.
- The verifier itself: gateway vs backend, library config, multi-service token reuse.

## Signals to Watch (the tells)

- **`alg` the client can influence**, or a library known for RS/HS confusion.
- **`kid`** pointing at a file path, URL, or DB lookup → path traversal / SQLi / key injection.
- **`jku` / `x5u`** (URL to key set) → if the host is attacker-influenceable or SSRF-reachable, you supply the key.
- **`jwk`** embedded in the header → some verifiers trust the key *in the token*.
- **Short, guessable, or default HMAC secrets** (HS256 with `secret`, `changeme`, a framework default).
- **Same token accepted by multiple services/subdomains** → aud/iss not enforced.
- **No visible rotation / long `exp`** → replay window.
- **ID token used where an access token belongs**, or refresh token accepted at a resource endpoint.

## Discovery / Methodology

1. **Pick a protected oracle** (401 without token, 200 with). This binary confirms everything.
2. **Baseline:** valid token → 200; none/garbage → 401. If no token → 200, that's a bigger finding: report and stop.
3. **Fingerprint:** decode header/claims; note alg, kid, jku/x5u, iss, aud, exp; fetch the JWKS.
4. **Run ONE probe at a time**, each a single crafted token.
5. **Confirm on the oracle** (see auth-jwt-triage). Stop at first accepted forgery — prove acceptance, don't ride it to full ATO.

## Exploitation Chains

### Chain 1 — alg:none / unsigned
1. Set header `alg` to `none` (and case variants `nOnE`), drop the signature, keep/escalate claims.
2. Send to the oracle.
3. **Confirm:** accepted → the verifier trusts an unsigned token. Critical.

### Chain 2 — Algorithm confusion (RS256 → HS256)
1. Obtain the server's RSA **public** key (JWKS, TLS cert, or a `/jwks.json`).
2. Re-sign the token with HS256 using the public key **as the HMAC secret**.
3. **Confirm:** accepted → the verifier used the public key as a symmetric secret. Forge arbitrary claims.

### Chain 3 — kid injection
1. `kid` feeds a file read or SQL query.
2. Point `kid` at a file whose contents you control/know (e.g., a predictable static file → sign with that), or inject SQL to return an attacker-known key.
3. **Confirm:** token signed with the coerced key is accepted.

### Chain 4 — jku / x5u key injection (often via SSRF)
1. `jku` points at a JWKS URL the server fetches.
2. Host your own JWKS (or use an SSRF/open-redirect/allowlist-bypass to reach an attacker-controlled response), sign with your private key.
3. **Confirm:** server fetches your key set and accepts your signature.

### Chain 5 — Signature not verified (tamper)
1. Flip a claim (`sub`, `role`, `isAdmin`, `tenant`) without re-signing.
2. **Confirm:** accepted → signature isn't checked at all.

### Chain 6 — Weak HMAC secret → forgery
1. Capture an HS256 token; crack the secret offline (hashcat mode 16500 / jwt_tool).
2. Forge a token with escalated claims signed with the recovered secret.
3. **Confirm:** accepted. (Offline cracking only; no online brute force against the target.)

### Chain 7 — Claim-scope / cross-service reuse
1. A token minted for service/audience A is replayed against service B.
2. Or `aud`/`iss` is weak/empty and accepted broadly.
3. **Confirm:** B accepts A's token → missing audience/issuer binding.

### Chain 8 — Expiry / token-type confusion
1. Replay an expired token (`exp` not enforced); or use an ID token where an access token is required; or a refresh token at a resource endpoint.
2. **Confirm:** accepted → lifetime/type not enforced.

## Techniques

- **Tooling:** `jwt_tool` (tamper/sign/attack modes), `hashcat` (offline secret crack), Burp JWT Editor (alg confusion, embedded jwk, kid).
- **Public-key sourcing** for RS→HS: JWKS `n`/`e` → reconstruct PEM; or pull from the TLS chain if reused.
- **One probe per token;** keep claims minimal and reversible; never chain into real ATO on a third party.

## Bypass Methods

- **Verifier differential:** gateway validates, backend trusts decoded claims (`X-User`/`X-Role` derived from the token) — target the backend.
- **Multi-issuer trust:** one of several accepted issuers/keys is attacker-influenceable.
- **JWKS cache/rotation window:** a stale or swapped key accepted during rotation.
- **Transport split:** one route validates, another (mobile/GraphQL) accepts the same token unchecked.
- **`none` listed as supported** but expected to be rejected — test it anyway.

## Validation

- **Confirmed:** a token the server must reject was accepted on an authenticated oracle (200 / elevated claim honored), with the specific defect named. See auth-jwt-triage for the full gate.
- Never promote on claim readability or "JWT is used" — tokens are readable by design.
- **Evidence + redaction:** the defect, the crafted header/claim diff (redacted), the oracle's accept response. Never output full valid tokens/keys; raw material to a debug artifact, redacted.

## Scope & Safety

- One probe at a time; stop at first confirmed acceptance — do not proceed to full account takeover.
- Offline secret cracking only; no online brute force, stuffing, or spraying.
- Never exfiltrate or persist real tokens/keys; redact everywhere including logs.
- Respect scope, rate limits, test windows; stop and flag if authorization is ambiguous.
