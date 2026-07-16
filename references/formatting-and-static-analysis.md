# Formatting & Static Analysis

## Language & build tool

| | Recommendation |
|---|---|
| Language | Kotlin (current stable major), or Java LTS for justified exceptions |
| Build | Gradle Kotlin DSL (`build.gradle.kts`) or Maven — pick one per service, don't mix |
| Framework | Latest stable Spring Boot 3.x/4.x |

Don't introduce a second build tool or a new JVM language into an existing service without a deliberate,
documented decision.

## Formatting — `.editorconfig`

Every service **must** have a root `.editorconfig` so formatting is identical across every service in the
org:

```ini
# EditorConfig is awesome: https://EditorConfig.org
root = true

[*]
charset = utf-8
indent_style = space
indent_size = 4
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
max_line_length = 240

[*.{yml,yaml}]
indent_size = 2

[*.md]
indent_size = 2
```

`max_line_length` is a hard ceiling, not a target — prefer lines under ~120 characters for readability.

## Static analysis

**Static analysis** = tools that read the source code (without running it) and flag problems: bad
formatting, style violations, convention breaks, and — with the right tool — actual bug patterns.
Catching these automatically means code review can focus on logic instead of spaces and naming.

The rule: **every service runs a linter, it runs in CI, and the build fails on violations.**

### Kotlin — ktlint (formatting) + detekt (smells & complexity)

ktlint enforces the official Kotlin style guide and formatting, and **auto-fixes** most issues.

```kotlin
// build.gradle.kts
plugins {
    id("org.jlleitschuh.gradle.ktlint") version "12.1.1"
}

tasks.check { dependsOn("ktlintCheck") }
```

ktlint has **no config file of its own — it reads `.editorconfig`**:

```ini
[*.{kt,kts}]
ktlint_code_style = ktlint_official
ktlint_standard_no-wildcard-imports = enabled
ktlint_standard_final-newline = enabled
```

Day-to-day:

```bash
./gradlew ktlintFormat    # auto-fix formatting before committing
./gradlew ktlintCheck     # verify formatting (what CI runs)
```

ktlint covers formatting and style only. For **complexity, code smells, and bug-prone patterns**, add
**detekt**:

```kotlin
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.7"
}

detekt {
    config.setFrom(files("$rootDir/config/detekt/detekt.yml"))
    buildUponDefaultConfig = true   // use detekt defaults; the yml only overrides what you change
}
```

```bash
./gradlew detektGenerateConfig   # writes config/detekt/detekt.yml once, commit it
./gradlew detektBaseline         # on an existing service: only new issues fail the build
```

**Warning:** detekt's `detekt-formatting` ruleset embeds ktlint. Use **either** standalone ktlint **or**
detekt-formatting for formatting — not both.

### Java — Spotless + Checkstyle (formatting & style) + PMD/SpotBugs (bugs)

| Tool | Job | Auto-fix? |
|---|---|---|
| **Spotless** (`google-java-format`) | Formatting | Yes — `mvn spotless:apply` |
| **Checkstyle** | Style + convention rules | No — reports only |
| **PMD** | Code smells, unused code, complexity | No — reports only |
| **SpotBugs** | Actual bug patterns (null derefs, resource leaks, etc.) | No — reports only |

Checkstyle alone only catches *style*, not *bugs* — pair it with PMD and/or SpotBugs the same way ktlint
pairs with detekt on the Kotlin side. Bind all of them to the `verify` phase so `mvn verify` (and CI)
**fails the build** on violations:

```xml
<plugin>
  <groupId>com.diffplug.spotless</groupId>
  <artifactId>spotless-maven-plugin</artifactId>
  <configuration>
    <java><googleJavaFormat/></java>
  </configuration>
  <executions>
    <execution>
      <goals><goal>check</goal></goals>
      <phase>verify</phase>
    </execution>
  </executions>
</plugin>
<!-- plus maven-checkstyle-plugin / maven-pmd-plugin / spotbugs-maven-plugin, all bound to verify -->
```

Day-to-day:

```bash
mvn spotless:apply    # auto-fix formatting before committing
mvn verify            # run all checks (what CI runs)
```

### Shared configuration (one ruleset for all services)

Define the ruleset once and reuse it, so every service is checked against the identical standard:

- **Kotlin:** the `.editorconfig` at the repo root is the shared ruleset — keep the `[*.{kt,kts}]` block
  identical across services (ideally distributed via a Gradle convention plugin published internally).
- **Java:** publish the shared `checkstyle.xml` / PMD ruleset as a small artifact and reference it from
  the plugin config in every service.

### Optional — a unified quality gate (SonarQube / SonarLint)

If the org wants one dashboard across languages instead of per-tool reports, add **SonarQube** (server
analysis, quality gates in CI) with **SonarLint** (the same rules surfaced live in the IDE). It overlaps
with detekt/PMD/SpotBugs in places — treat it as a **replacement** for the "smells" layer, not an
additional one, to avoid triaging the same issue reported by two tools under different names.

## Naming conventions

### Packages

Base package is one root per service, all lowercase, no underscores, no camelCase:

```
com.example.user
com.example.catalog
com.example.notification
```

### Classes

| Suffix | Meaning |
|---|---|
| `*Controller` | REST controller (interface); `*ControllerImpl` if split |
| `*Service` / `*ServiceImpl` | Business logic — interface + implementation |
| `*Repository` | Data access |
| `*Mapper` | DTO ↔ entity mapping |
| `*Request` / `*Response` | API DTOs (see [`api-design.md`](./api-design.md)) |
| `*Config` / `*Configuration` | Spring `@Configuration` classes |
| `*Consumer` | Message / AMQP consumers |
| `*Exception` | Custom exceptions |

### Members

- Functions and properties: `lowerCamelCase`. Booleans read as predicates: `isActive`, `hasAccess`.
- Constants: `UPPER_SNAKE_CASE`. Types: `UpperCamelCase`.
- Names reveal intent; avoid abbreviations beyond well-known ones (`id`, `dto`, `url`).

## Package layout

Two layouts are common; pick one per service and apply it consistently.

### Option A — classic layered (small/simple services)

```
com.example.<service>
├── controller/
├── service/
├── repository/
├── dto/
├── model/
└── util/
```

Simple, but as a service grows every layer becomes a dumping ground with no enforced boundary between
feature areas.

### Option B — modular (Spring Modulith), classic MVC *inside* each module

For services with multiple feature areas, structure by **module** (one per feature), and use the classic
MVC layout *inside* each module. The module's base package is its public API; sub-packages are internal
and Spring Modulith enforces that other modules can't reach into them.

```
com.example.<service>
├── <ServiceName>Application.kt
├── order/                       # a Modulith module (feature area)
│   ├── OrderApi.kt              # PUBLIC — the only type other modules may call
│   ├── OrderData.kt             # PUBLIC — data the API exposes across modules
│   ├── controller/              # internal: web layer
│   ├── service/                 # internal: business logic (implements OrderApi)
│   ├── repository/              # internal: data access
│   ├── dto/                     # internal: web request/response DTOs
│   ├── model/                   # internal: entities
│   └── util/                    # internal: module-local helpers
├── invoice/                     # another module, same layout
└── common/                      # shared config, base utils (cross-cutting only)
```

- Modules are **cohesive and loosely coupled**; a module owns its controllers, services, repositories and
  entities and does not share them directly.
- `common/` holds only genuinely cross-cutting infrastructure (config, base utilities) — not business
  logic.
- We deliberately do **not** use DDD layering (`api`/`application`/`domain`/`infrastructure`) inside a
  module — it adds indirection most services don't need; classic MVC inside a Modulith module gives the
  same isolation with less ceremony.

### Communicating between modules

**Never reach into another module's internals** (its `service`, `repository`, `model`). Expose an **API
service at the top level of that module** and depend only on it.

```kotlin
// order/OrderApi.kt  — PUBLIC contract at the module base package
interface OrderApi {
    fun findOrder(orderId: String): OrderData?
}

// order/OrderData.kt — PUBLIC cross-module data (not the internal entity)
data class OrderData(val id: String, val total: BigDecimal, val status: String)

// order/service/OrderService.kt — INTERNAL implementation
@Service
class OrderService(
    private val repository: OrderRepository,   // internal to this module
) : OrderApi {
    override fun findOrder(orderId: String): OrderData? =
        repository.findById(orderId)?.let { OrderData(it.id, it.total, it.status) }
}
```

```kotlin
// invoice/service/InvoiceService.kt — another module depends only on OrderApi
@Service
class InvoiceService(
    private val orderApi: OrderApi,   // the public API — NOT OrderRepository / Order
) {
    fun createInvoice(orderId: String) {
        val order = orderApi.findOrder(orderId) ?: error("order not found")
        // ...
    }
}
```

For decoupled, asynchronous flows, publish a Spring Modulith **application event** instead of a direct
call (`ApplicationEventPublisher` → `@ApplicationModuleListener`).

### Enforce the module boundary with a test

```kotlin
@Test
fun `verifies modulith structure`() {
    ApplicationModules.of(Application::class.java).verify()
}
```

This test **must stay green** — it fails the build if a module accesses another module's internal
packages. If it fails, fix it by routing the access through the target module's public API — never widen
visibility or disable the check.

## Dependency injection

**Constructor injection only.**

```kotlin
class AccountService(
    private val repository: AccountRepository,
    private val mapper: AccountMapper,
)
```

Do **not** use field injection (`@Autowired` on fields) or setter injection — both hide dependencies and
break testability (you can't construct the class in a unit test without Spring).

If Spring reports a **circular dependency**, that's a design smell, not a wiring problem to work around
with `@Lazy` — extract the shared behaviour into a third collaborator, or invert one direction into an
event.

## Logging

- One logger per class; log at boundaries and on error paths, never inside tight loops.
- **Parameterized logging, not string concatenation** — `log.info("loaded {} records", count)`, not
  `log.info("loaded " + count + " records")`. Concatenation runs even when the log level is disabled;
  parameterized calls skip formatting entirely.
- Do not log-and-rethrow the same exception — handle it once, at the boundary.
- Never log secrets, tokens, or PII.
