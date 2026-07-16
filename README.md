# Kotlin / Java / Spring Boot Code Style — Claude Code Skill

A portable Claude Code skill that packages a full backend code-style standard for **Kotlin and Java
services built on Spring Boot**: clean code, formatting & static analysis, Spring Boot idioms, REST API
design, secure coding, testing, and documentation conventions.

It started as a generalized, extended version of an internal backend code-style guide, filled out with
additional industry best practices that weren't covered yet: Java records/sealed types, virtual threads,
Kotlin coroutines vs. blocking I/O, Resilience4j patterns, JPA entity `equals`/`hashCode` pitfalls, HikariCP
tuning, Actuator hardening, REST idempotency keys, and PMD/SpotBugs/SonarQube as bug-pattern analysis
alongside ktlint/detekt/Checkstyle.

## What's inside

```
kotlin-java-spring-boot-code-style/
├── README.md                              — this file
├── SKILL.md                               — the skill entry point (frontmatter + navigation)
└── references/
    ├── clean-code.md                      — function/class design, naming, Kotlin & Java idioms,
    │                                          collections, domain modeling, error handling,
    │                                          transactions, pagination, date/time, concurrency
    ├── formatting-and-static-analysis.md   — .editorconfig, ktlint/detekt, Spotless/Checkstyle/
    │                                          PMD/SpotBugs, package layout, DI, logging
    ├── spring-boot-practices.md            — config properties, transactions, caching, resilience,
    │                                          HTTP clients, connection pools, virtual threads,
    │                                          coroutines, Actuator, bean lifecycle
    ├── api-design.md                       — REST conventions, DTOs, RFC 9457 errors, OpenAPI +
    │                                          linting + breaking-change detection, idempotency
    ├── security.md                         — authn/authz, input validation, injection, secrets,
    │                                          dependency scanning, security testing
    ├── testing.md                          — frameworks, unit vs. integration, naming, coverage
    ├── documentation.md                    — doc types, KDoc/Javadoc rules
    └── checklist.md                        — everything above as one actionable checklist
```

`SKILL.md` is intentionally short — it states the non-negotiables and points to the right reference file
per topic, so Claude only loads the detail it needs for the task at hand instead of the whole standard on
every turn.

## Installing it

Repo: [github.com/OrestKlymko/clean-code-kotlin-java-skill](https://github.com/OrestKlymko/clean-code-kotlin-java-skill)

```bash
git clone git@github.com:OrestKlymko/clean-code-kotlin-java-skill.git ~/.claude/skills/kotlin-java-spring-boot-code-style
```

Or drop the folder manually into a skills directory Claude Code reads from:

- **Personal, all projects:** `~/.claude/skills/kotlin-java-spring-boot-code-style/`
- **This project only:** `<project-root>/.claude/skills/kotlin-java-spring-boot-code-style/`

No build step, no dependencies — it's just markdown. Restart Claude Code (or start a new session) and the
skill shows up in the available-skills list.

## How it triggers

The `description` in `SKILL.md`'s frontmatter is what Claude matches against your request. As written, it
fires for: writing new Kotlin/Java/Spring Boot code, reviewing a diff in such a codebase, scaffolding a new
service, or asking for a code-style/checklist reference. You can also invoke it explicitly by name.

## Adapting it to your org

The examples use a generic package root (`com.example.<service>`) and a generic shared library name
(`platform-commons`) instead of any company-specific naming. Before treating this as your team's binding
standard:

1. Swap the generic names for your actual conventions (package root, shared library name, identity
   provider, artifact repository).
2. Adjust numbers that are genuinely team decisions, not universal truths (the 80% coverage floor, the
   240-char line-length ceiling, `v1`-style path versioning) to whatever your team has actually agreed on.
3. Delete or rewrite any section that conflicts with an existing house standard — this skill is a strong
   default, not a mandate.

## Scope

This skill governs **service-side Kotlin/Java/Spring Boot code**. It does not cover frontend code style,
infrastructure/GitOps conventions, or product/process topics (SDLC, ticket workflow) — those belong in
separate skills or your team's own documentation.
