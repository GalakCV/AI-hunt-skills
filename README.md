# AI-hunt-skills

Agent skills for authorized web bug-bounty / offensive-security testing, written for [Strix](https://github.com/usestrix/strix)-style agents (`SKILL.md` format: YAML frontmatter + markdown body, loaded by filename).

Two layers, meant to load together:

- **`skills/`** — the attack catalog. Per vulnerability class: mindset, the *tells* to watch in traffic, end-to-end exploitation chains, concrete techniques, and bypass methods.
- **`triage/`** — an optional confirmation gate. The low-false-positive workflow for each class: one invariant per test, minimal non-destructive PoC, confirmation on the *authoritative source* (not the HTTP 200), confidence levels (confirmed / high / medium / low), and mandatory secret redaction.

## Skills

| Class | Attack skill | Triage/confirm |
|---|---|---|
| Business logic | `skills/business-logic.md` | `triage/business-logic-triage.md` |
| IDOR / BOLA | `skills/idor.md` | `triage/idor-triage.md` |
| Race conditions | `skills/race-conditions.md` | `triage/race-conditions-triage.md` |
| JWT / OIDC | `skills/authentication-jwt.md` | `triage/auth-jwt-triage.md` |
| Mass assignment | `skills/mass-assignment.md` | `triage/mass-assignment-triage.md` |
| GraphQL | `skills/graphql.md` | reuse the class triage (idor / business-logic / …) |

## Usage

Skills load by the `name` in their frontmatter. Load an attack skill alongside its triage gate, e.g.:

```python
create_agent(task="Test checkout for logic flaws",
             skills="business-logic-exploitation,business-logic-triage")
```

The `skills/` files reuse the native class names (`idor`, `race-conditions`, …); drop one into a Strix `skills/<category>/` directory to override the built-in with this richer version, or keep the `-exploitation` name to load both.

