---
name: kotlin-java-spring-boot-code-style
description: Best-practice code style guide for writing, reviewing, and refactoring Kotlin and Java code in Spring Boot services — clean code, formatting/static analysis, Spring Boot idioms, REST API design, security, testing, and documentation. Use whenever writing, reviewing, or refactoring backend Kotlin/Java/Spring Boot code, setting up a new service, or defining a team code-style standard.
version: 1.0.0
---

# Kotlin / Java / Spring Boot Code Style

A reference standard for backend services built with **Kotlin or Java + Spring Boot**. It covers the
decisions a linter can't check: design, naming, error handling, API shape, security, testing, and the
Spring Boot idioms that separate a maintainable service from a fragile one.

This is a **standard to apply**, not trivia to recite. When writing code, hold it to these rules. When
reviewing code, use it as the review criteria. When bootstrapping a new service, start from
[`references/checklist.md`](references/checklist.md).

## When to use this skill

- Writing new Kotlin/Java/Spring Boot code (services, controllers, entities, tests).
- Reviewing a diff or PR in a Kotlin/Java/Spring Boot codebase.
- Scaffolding a new Spring Boot service or module.
- Deciding how to structure, name, test, or secure something and no local convention says otherwise.

## Non-negotiables

These apply regardless of which reference file is loaded:

1. **No secrets in code, logs, or version control.** Config from env/secret store only.
2. **Every endpoint has an explicit authorization rule.** Deny by default.
3. **Validate all external input at the boundary.** Allow-list, don't deny-list.
4. **Never leak stack traces or internal detail to a client.** Errors go through one central handler.
5. **No `!!` in Kotlin, no swallowed exceptions in either language.** Fail fast, fail loud.
6. **Constructor injection only.** No field/setter injection.
7. **A function/class does one thing.** If "and" describes it, split it.

## Map of this skill

| Topic | File | Covers |
|---|---|---|
| Clean code & language idioms | [`references/clean-code.md`](references/clean-code.md) | Function/class design, naming, comments, Kotlin & Java idioms (null-safety, records, sealed types), collections, domain modeling, error handling, transactions, pagination, date/time, resource management, concurrency |
| Formatting & static analysis | [`references/formatting-and-static-analysis.md`](references/formatting-and-static-analysis.md) | `.editorconfig`, ktlint/detekt, Spotless/Checkstyle/PMD/SpotBugs, SonarQube, naming conventions, package/module layout, dependency injection, logging |
| Spring Boot practices | [`references/spring-boot-practices.md`](references/spring-boot-practices.md) | Configuration properties, transactions, caching, resilience (timeouts/retry/circuit breaker), HTTP clients, connection pools, virtual threads, coroutines vs. blocking I/O, Actuator, bean lifecycle |
| API design | [`references/api-design.md`](references/api-design.md) | REST conventions, DTOs, RFC 9457 error bodies, OpenAPI + linting + breaking-change detection, pagination, idempotency |
| Security | [`references/security.md`](references/security.md) | AuthN/AuthZ, input validation, injection prevention, secrets, dependency scanning, security testing |
| Testing | [`references/testing.md`](references/testing.md) | Frameworks, unit vs. integration, naming, structure, coverage, fixtures |
| Documentation | [`references/documentation.md`](references/documentation.md) | Doc types, KDoc/Javadoc rules, what to document vs. what to leave to code |
| Full checklist | [`references/checklist.md`](references/checklist.md) | Everything above rolled into one actionable list for a new service or a PR review |

## How to apply it

- **Writing a feature** → `clean-code.md` + `spring-boot-practices.md` + `testing.md` are the day-to-day
  reference. Pull in `api-design.md` if it touches a REST endpoint, `security.md` if it touches auth or
  external input.
- **Reviewing a PR** → skim `checklist.md`; treat any unchecked non-negotiable as a blocking comment.
- **Scaffolding a new service** → work through `checklist.md` top to bottom before writing business logic.
- **Unsure which language idiom applies** → `clean-code.md` has parallel Kotlin/Java sections; use the one
  matching the file you're editing, and don't mix idioms of one language into the other (e.g. don't
  write Kotlin-style builders in Java when a `record` fits, don't fight Java's `Optional` conventions
  into Kotlin where nullable types already do the job).

## Adapting to a specific org

The examples use a generic package root (`com.example.<service>`) and a generic shared library name
(`platform-commons`). Swap these for your own conventions — the *pattern* (one base package per service,
one shared plumbing library, constructor injection, etc.) is what to keep, not the literal string.
