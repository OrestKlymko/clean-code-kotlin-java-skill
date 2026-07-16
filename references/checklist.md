# Service / PR Checklist

Use this when **creating a new service** or **reviewing** one against the standard. It rolls up every
rule in this skill into a single actionable list.

## Project & build

- [ ] Kotlin/Java + Gradle Kotlin DSL or Maven, latest stable Spring Boot
- [ ] One base package per service (lowercase, no underscores)
- [ ] Package layout chosen deliberately (classic layered, or Spring Modulith modules with classic MVC
      inside) — see [`formatting-and-static-analysis.md`](./formatting-and-static-analysis.md#package-layout)
- [ ] Cross-module calls go through a module's top-level API only — no reaching into internals
- [ ] Root `.editorconfig` in place

## Quality gates (fail the build)

- [ ] ktlint + detekt (Kotlin) / Spotless + Checkstyle + PMD or SpotBugs (Java) wired into the build's
      verification task
- [ ] JaCoCo with an enforced instruction-coverage threshold wired into the build
- [ ] Architecture test verifying module/layer boundaries (Spring Modulith `ApplicationModules.verify()`
      or ArchUnit) — kept green, never disabled

## Clean code

- [ ] Small, single-purpose functions; guard clauses over nesting
- [ ] Immutability by default (`val`/`record`); no `!!`, no unchecked `Optional.get()`
- [ ] `@Transactional` in the service layer; read-only query paths use `@Transactional(readOnly = true)`
- [ ] Large/growing result sets use **keyset pagination**, not deep offset
- [ ] Message consumers idempotent; risky `POST` endpoints support an idempotency key
- [ ] JPA entities have deliberate `equals`/`hashCode` (not the `data class`/`@Data`/record default)

## API

- [ ] REST paths `/api/v1/…`; `Request`/`Response` DTOs; constructor injection
- [ ] `@ControllerAdvice` → RFC 9457 `ProblemDetail` error handling
- [ ] Entities never exposed over the wire (always mapped to DTOs)
- [ ] `springdoc` OpenAPI exposed **and** the generated spec committed to the repo
- [ ] OpenAPI spec linted in CI (vacuum/Spectral) and checked for breaking changes (oasdiff)

## Spring Boot practices

- [ ] Config bound via typed, validated `@ConfigurationProperties` — no scattered `@Value`
- [ ] Outbound calls have explicit timeouts; retries only on idempotent-safe operations
- [ ] Connection pool (HikariCP) sized deliberately, with `connectionTimeout` and leak detection set
- [ ] Actuator exposes only what's needed; `env`/`heapdump`/`threaddump` locked down in production
- [ ] No heavy work in bean constructors; singleton beans are stateless or thread-safe

## Security

- [ ] Deny-by-default; every endpoint has a `@PreAuthorize`
- [ ] Object-level authorization where the resource is user/tenant-scoped
- [ ] All input `@Valid`-ated; parameterized queries only
- [ ] No secrets in code/config/logs; caller identity from the security context
- [ ] Dependency/container scan enabled in CI

## Testing

- [ ] JUnit 5; MockK (Kotlin) / Mockito (Java)
- [ ] Testcontainers integration tests (`*IT`) for any repository / query / migration logic
- [ ] Negative-authorization and validation tests

## Shared library usage

- [ ] Reuse the org's shared platform library for cross-cutting plumbing instead of copying code
- [ ] Only the modules this service actually needs are enabled
- [ ] No business logic pushed into the shared library; no service code the shared library depends on

## Documentation

- [ ] `README.md` (description, architecture, run locally, config, API, testing)
- [ ] `CHANGELOG.md`
- [ ] Feature docs for any non-trivial, shipped behaviour

## CI/CD

- [ ] Pipeline runs tests + coverage + lint + dependency scan, and fails on any gate
- [ ] Versioned release process (version bump, changelog, tag)
