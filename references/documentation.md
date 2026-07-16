# Documentation Standards

Consistency beats freedom. Pick the doc type, use its skeleton, fill the sections in order — don't invent
a local layout per author.

## Where documentation lives

| Type | Where | Answers |
|---|---|---|
| **Service** | `README.md` in the service repo | What is this service, how do I run/configure it |
| **Feature** | team docs site / wiki | How does a shipped feature work, and why |
| **Analysis / design** | team docs site / wiki | A problem, the options considered, the decision |
| **Handover** | team docs site / wiki | Operational knowledge for whoever picks up the work |
| **API** | committed OpenAPI spec | What endpoints exist, request/response shapes |
| **Code** | KDoc / Javadoc | Why non-obvious code does what it does |

Rule of thumb:

- **Feature** = "how does X work *today*" (present tense, describes shipped behaviour).
- **Analysis** = "should we build X, and how" (a decision record — problem → decision).
- **Handover** = "here's what you need to continue this work" (state, next steps, open questions).

## Formatting rules

- **ATX headings only** (`#`, `##`, `###`). Avoid Setext-style headings (title underlined with `===` or
  `---`) — mixing the two styles across a doc set is the single biggest source of inconsistency.
- **One H1 per page.**
- **Tables** for anything with 3+ parallel items (params, options, statuses).
- **Diagrams** (Mermaid or equivalent) for any flow worth more than two sentences.
- **Admonitions/callouts** for warnings and notes, not bold-shouting in prose.

## Service `README.md`

Required in every service repo. Minimum sections:

```markdown
# <service-name>

One-paragraph description: what this service owns and its place in the platform.

## Architecture
Key modules / bounded contexts, main dependencies (DB, broker, other services).

## Running locally
Prerequisites (JDK, Docker), how to start (`./gradlew bootRun`), link to the dev stack.

## Configuration
Important env vars / config properties and what they do.

## API
Link to the Swagger UI URL and the committed OpenAPI spec.

## Testing
How to run unit and integration tests.
```

Keep a `CHANGELOG.md` per service, current with each release.

## Feature doc skeleton

For shipped, non-trivial or cross-cutting behaviour. Present tense — describes what the system does now.

```markdown
One-paragraph summary: what this feature does and who/what triggers it.

## Overview
Bullet list of what it provides. Key concepts / terms.

## How it works
(diagram)
1. Step one …
2. Step two …

## Configuration        (optional)
Relevant properties / feature flags and what they do.

## Edge cases & gotchas
- …

## Related
Links to analysis docs, API conventions, other features.
```

## Analysis / design doc skeleton

A decision record — problem → discussion → solution, in this fixed order so a reader can stop at
**Decision** and know the outcome; **Solution** is for whoever implements it.

```markdown
## Problem
What are we solving and why now. The pain, the trigger, the constraint.

## Context
Current state, relevant requirements, constraints, assumptions.

## Options considered
For each option: what it is, pros, cons. A table works well.

## Decision
The chosen option and the reasoning. State it plainly.

## Solution
How the decision is realised — design, data model, flow, affected services.

## Consequences
Trade-offs accepted, follow-up work created, what this rules out later.
```

## Code-level — KDoc / Javadoc

The rule is **not** "document everything":

- **Do** document: public APIs of shared libraries; non-obvious algorithms; the *why* behind a
  workaround; anything that surprised you while writing it.
- **Don't** write doc comments that restate the signature (`// gets the user` on `fun getUser()`).
- Prefer clear names and small functions over comments that compensate for unclear code.
- Exported types of a **shared library** are fully documented — consumers see only the API, not the
  source context.

```kotlin
/**
 * Computes the content hash of a record over its mapped attributes only.
 *
 * Unmapped attributes are excluded so unrelated field changes don't register as a change.
 */
fun contentHash(record: Record, mapping: AttributeMapping): String { /* ... */ }
```
