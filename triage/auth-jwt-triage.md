---
name: auth-jwt-triage
description: Operational playbook for triaging and confirming JWT/OIDC flaws with high signal and low false positives — captures tokens from recon/traffic, runs non-destructive validation probes (alg confusion, kid, aud/iss, expiry), and confirms only when a forged or tampered token is accepted on an authenticated endpoint. Works both recon-fed/assisted and fully autonomous, with strict secret redaction. Load alongside the authentication-jwt skill: that one is the technique catalog, this one is the execution and confirmation workflow. Use when triaging token handling or deciding whether a suspected JWT flaw is real.
---

# JWT / OIDC — Triage & Confirmation

A JWT flaw is proven only when a token the server should reject is accepted on a protected action — not when the library "looks" misconfigured. This skill runs the minimal set of non-destructive probes and confirms on a real authenticated endpoint, with hard rules on redacting the secrets you handle.

Pair with the `authentication-jwt` skill for the catalog (algorithm families, OIDC flows, chaining). This skill governs *how you run and confirm*.

## Attack Surface

- **Intake from recon first.** Pull the access token, its header/claims (alg, kid, typ, iss, aud, exp, sub, scope/roles), the issuer's JWKS URL, and every endpoint that consumes the token. Note which endpoints are authenticated vs public — your confirmation oracle must be authenticated.
- Prioritize endpoints that gate privilege or data (role-sensitive, tenant-scoped, admin).
- A decoded token is not a finding. The surface is the *verifier's* acceptance behavior.

## Methodology

1. **Pick a protected oracle.** One endpoint that returns 401/403 without a valid token and 200 with one. This binary is your confirmation.
2. **Baseline:** valid token → 200; no/garbage token → 401. If the endpoint 200s with no token, that is a separate (and bigger) finding — report it and stop.
3. **Run ONE probe at a time** (see Techniques), each a single tampered token.
4. **Confirm on the oracle.** The flaw is real only if the tampered/forged token yields 200 (or the elevated claim takes effect). Any 401/403 means the verifier held.
5. **Gate and stop.** Do not chain to a real account takeover; prove acceptance and report.

### Autonomous vs assisted

- **Assisted (recon-fed):** output each crafted token (redacted), the oracle endpoint, and the expected accept/reject. Do not send.
- **Autonomous:** send the probe to the oracle, read accept/reject, report the gated finding.

Do not perform credential stuffing, brute force, or password spraying — those are usage-policy and scope issues, not JWT validation tests.

## Techniques

Each probe = one tampered token, confirmed on the oracle:

- **alg:none / unsigned:** set alg to none (and variants nOnE), strip signature; accepted = critical.
- **Algorithm confusion (RS→HS):** re-sign with the public key as an HMAC secret; accepted = the verifier confused asymmetric for symmetric.
- **kid manipulation:** point kid at an injectable path/URL/SQL to coerce an attacker-known key; accepted = key-selection flaw.
- **Signature not verified:** flip a claim (sub, role) without re-signing; accepted = no verification at all.
- **Claim scope:** weak/empty/mismatched aud or iss accepted across services → cross-service token reuse.
- **Expiry/nbf:** replay an expired token; accepted = lifetime not enforced.
- **Confusion of token types:** use an ID token where an access token is required, or a refresh token at a resource endpoint.

## Bypass Methods

- Verifier differential: gateway validates but a backend service trusts the decoded claims (X-User / X-Role from a parsed token) — target the service.
- Multiple accepted issuers/keys where one is attacker-influenced.
- JWKS cache/rotation window accepting a stale or swapped key.
- Transport split: one route validates, another (mobile/GraphQL) accepts the same token unchecked.

## Validation

### Confidence gating (report nothing above its evidence)

- **Confirmed:** a token the server must reject was accepted on an authenticated oracle (200 / elevated claim honored), reproduced ≥2×, with the specific defect named (alg:none, RS/HS, unverified signature, etc.).
- **High:** structural weakness strongly implied (e.g., HS/RS confusion token returns 200 once) but not re-confirmed — re-run to confirm.
- **Medium:** permissive config visible (weak alg allowed, long expiry) with no accepted forged token yet — needs the oracle test.
- **Low / informational:** decodable token, readable claims, missing best-practice header. **Not a vulnerability.**

Never promote on "JWT is used" or claim readability — tokens are not encrypted by default; readability is expected.

### Evidence & redaction

- **Redaction is mandatory and strict.** Never output a full valid token, cookie, private key, or the JWKS private material. Show only the tampered header/claim diff and the signature method; mask the payload and signature (first/last few chars at most).
- Minimal evidence: the one defect, the crafted token's header/claim change (redacted), and the oracle's accept response. Raw tokens/traffic → debug artifact by ID, redacted.

### False positives to drop on sight

- Readable (base64) claims treated as "disclosure".
- Permissive config with the verifier still rejecting forged tokens.
- Short-lived tokens flagged purely on lifetime with no acceptance of a forged one.
- `none` listed as supported but rejected at verification.

## Safety Caps (both modes)

- One probe (one tampered token) at a time; stop at first confirmed acceptance — do not proceed to full ATO.
- No credential stuffing, brute force, or password spraying.
- Never exfiltrate or persist real tokens/keys; redact everywhere, including logs.
- Respect scope, rate limits, and test windows; stop and flag if scope/authorization is ambiguous.
