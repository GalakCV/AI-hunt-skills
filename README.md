# AI-hunt-skills

Agent skills for authorized web bug-bounty / offensive-security testing, in the [Strix](https://github.com/usestrix/strix) skill format (`<name>.md`: YAML frontmatter with `name`/`description` + markdown body).

Two layers, meant to load together:

- **`skills/`** — the attack catalog. Per vulnerability class: mindset, the *tells* to watch in traffic, end-to-end exploitation chains, concrete techniques, and bypass methods.
- **`triage/`** — an optional confirmation gate. The low-false-positive workflow for each class: one invariant per test, minimal non-destructive PoC, confirmation on the *authoritative source* (not the HTTP 200), confidence levels (confirmed / high / medium / low), and mandatory secret redaction.

These are richer, exploitation-focused versions of several Strix built-ins. They use **distinct names** (`-exploitation` suffix) so they live **alongside** the native skills, not shadowing them — load whichever you want.

## Skills

Load a skill by its **filename without `.md`** (this is the name Strix uses — the `name:` in the frontmatter matches it).

| Class | Attack skill (load name) | Triage/confirm (load name) |
|---|---|---|
| Business logic | `business-logic-exploitation` | `business-logic-triage` |
| IDOR / BOLA (object-level) | `idor-exploitation` | `idor-triage` |
| BFLA (function-level) | `bfla-exploitation` | reuse `idor-triage` / `business-logic-triage` |
| Race conditions | `race-conditions-exploitation` | `race-conditions-triage` |
| JWT / OIDC | `authentication-jwt-exploitation` | `auth-jwt-triage` |
| Mass assignment | `mass-assignment-exploitation` | `mass-assignment-triage` |
| GraphQL | `graphql-exploitation` | reuse the class triage |
| JavaScript recon | `javascript-recon` | feeds the class skills (leads, not findings) |

## Install into Strix

Point Strix at this repo as an extra skill directory. The subfolders (`skills/`, `triage/`) become skill categories; filenames become the loadable names.

```python
from strix.skills import register_skill_dir
register_skill_dir("/path/to/AI-hunt-skills")
```

Then load by name (max 5 per agent), pairing an attack skill with its triage gate:

```python
create_agent(task="Test checkout for logic flaws",
             skills="business-logic-exploitation,business-logic-triage")
```

Names are unique across the native set, so they load bare (no `category/` prefix needed). If you ever hit an ambiguity error, qualify with the category, e.g. `skills/idor-exploitation`.

> Note: Strix selects a skill by its **filename**, not by the `name:` field. Every file here keeps the two identical to avoid surprises.

## Scope

For **authorized, in-scope** targets only. Every skill mandates minimal, reversible, non-destructive proofs of concept, respects rate limits / test windows / out-of-scope lists, and redacts tokens, credentials, and PII. Write/money/irreversible actions default to operator review rather than autonomous execution. Unauthorized testing is illegal — you are responsible for your authorization.
