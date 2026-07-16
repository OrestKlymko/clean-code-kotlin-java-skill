# Clean Code & Language Idioms

Formatting and CI gates make code *consistent*. This file makes it *maintainable* — the design decisions
a linter can't check. Rules of thumb, not laws; the goal is code the next person understands in one read.

## Functions & classes

- **Small, single-purpose.** A function does one thing; a class has one reason to change (SRP). If you
  need "and" to describe it, split it.
- **Short.** Prefer functions that fit on a screen. Extract private helpers with intention-revealing
  names rather than growing one long method.
- **Guard clauses over nesting.** Return early; don't pyramid `if`s.

  ```kotlin
  // avoid
  fun handle(order: Order?) {
      if (order != null) {
          if (order.isValid) { /* ... */ }
      }
  }

  // prefer
  fun handle(order: Order?) {
      if (order == null) return
      if (!order.isValid) return
      // ...
  }
  ```

- **Few parameters.** More than ~3–4 → wrap them in a request object / data class / record.
- **No flag arguments.** A `Boolean` param that switches behaviour = two functions.
- **Favor composition over inheritance.** Reach for an interface + delegation before a base class;
  inheritance couples subclasses to a parent's implementation details, not just its contract.
- **Design a class for extension deliberately, or close it off.** In Kotlin, classes are `final` by
  default — only `open` a class/member when subclassing is an intended use case, not an accident. In
  Java, mark classes `final` unless extension is designed for.

## Naming

- Names reveal intent: `overdueInvoices`, not `list2`. Booleans read as predicates (`isActive`,
  `hasAccess`).
- Same concept → same word everywhere. Don't mix `fetch`/`get`/`load`/`retrieve` for the same idea.

## Comments

Clean code is not comment-free code — it's code where every comment carries information the code
itself *cannot* express. Few comments in code, but not zero.

- **Why, never what.** A comment restating the line below is noise; a comment explaining a
  non-obvious reason is gold.

  ```kotlin
  // avoid — restates the code
  // increment retry counter
  retries++

  // prefer — explains what the code can't
  // upstream returns 409 on concurrent import — one retry is enough
  retryOnce { importRealm(realm) }
  ```

- Comment **non-obvious constraints**: ordering requirements, workarounds for upstream bugs (link the
  ticket), deliberate performance trade-offs.
- If you need a comment to explain *what* a block does — don't write the comment; extract a method
  with an intention-revealing name instead.
- **Public API gets KDoc/Javadoc.** Shared-library code documents contract, activation conditions, and
  configuration knobs — consumers can't read your intent from a call site. See
  [`documentation.md`](./documentation.md).
- **Configuration files are the exception — comment generously.** Properties, YAML, env files, and
  compose files can't self-document: no names, no types, no structure. Explain units, valid values,
  and why a value deviates from the default.
- **Always delete:** commented-out code, `TODO` without a ticket, changelog-style comments
  ("added by X on date") — version control remembers.

## Kotlin idioms

- **Immutability first.** `val` over `var`; `data class` for DTOs / value objects; return new copies with
  `copy()` instead of mutating.
- **Null-safety — never `!!`.** Use `?.`, `?:`, `requireNotNull(x) { "msg" }`, or model absence in the
  type. A `!!` is a latent NPE and a code-review red flag.
- **Model choices with `sealed` types / enums**, not string constants or magic numbers.

  ```kotlin
  sealed interface ImportResult {
      data class Success(val recordCount: Int) : ImportResult
      data class Failed(val reason: String) : ImportResult
  }
  ```

- Use `when` exhaustively over sealed types — the compiler enforces every case is handled.
- **Extension functions** for mapping/formatting, kept next to the type they extend so they're
  discoverable.
- **Named arguments + default parameter values instead of a builder.** Kotlin's calling convention
  already solves the "too many constructor params" problem a Java builder pattern exists for — don't
  port the builder pattern into Kotlin.
- **Value classes (`@JvmInline value class`)** to wrap a primitive with domain meaning at zero runtime
  cost — see [Domain modeling](#domain-modeling).
- Prefer **coroutines' structured concurrency** (`coroutineScope`, `supervisorScope`) over manually
  launching and tracking `Job`s — a scope guarantees children finish or are cancelled before it exits.

## Java idioms

- **Records for immutable data carriers** (Java 16+). A `record` gives you a constructor, accessors,
  `equals`/`hashCode`/`toString` for free — prefer it over a hand-rolled immutable POJO or a Lombok
  `@Value` class for DTOs and value objects.

  ```java
  public record AccountResponse(String id, String email, String displayName) {}
  ```

- **Sealed interfaces + pattern-matching `switch`** (Java 17 sealed types, Java 21 pattern matching) as
  the Java equivalent of Kotlin's `sealed` + `when`: model a closed set of outcomes so the compiler
  flags a missing branch.

  ```java
  sealed interface ImportResult permits Success, Failed {}
  record Success(int recordCount) implements ImportResult {}
  record Failed(String reason) implements ImportResult {}

  String describe(ImportResult result) {
      return switch (result) {
          case Success s -> "imported " + s.recordCount();
          case Failed f -> "failed: " + f.reason();
      };
  }
  ```

- **`var` for local variables** — use it when the right-hand side already makes the type obvious
  (`var users = new ArrayList<User>()`); avoid it when it would hide the type from the reader
  (`var result = process(input);` — what is `result`?).
- **`Optional` is a return type, not a field or parameter type.** Use it to signal "this method may have
  nothing to return"; don't store it on a field, pass it as a parameter, or call `.get()` without
  `isPresent()`/`orElseThrow` — that's just a slower `!!`.
- **Be deliberate with Lombok.** `@Getter`/`@Builder` on plain DTOs are fine; avoid `@Data` /
  `@EqualsAndHashCode` / `@ToString` on **JPA entities** — see
  [Entity `equals`/`hashCode`](#entity-equals-hashcode-and-tostring) below for why.
- **Builder pattern** (hand-written or Lombok `@Builder`) for objects with many optional parameters or
  that are constructed progressively — the Java equivalent of Kotlin named/default arguments.

## Constants & magic values

- **No magic numbers or strings** in logic. Name them — a `const val` / `static final` / enum reveals
  intent and gives one place to change.

  ```kotlin
  // avoid
  if (retries > 3) fail()
  // prefer
  const val MAX_RETRIES = 3
  if (retries > MAX_RETRIES) fail()
  ```

- Group related constants on a `companion object` or an enum, not scattered across the file.
- Configuration values (timeouts, URLs, limits) belong in externalized config, not as source constants —
  see [`spring-boot-practices.md`](./spring-boot-practices.md#configuration-properties).

## Collections

- **Return empty, never null.** A function returning a collection returns `emptyList()` on "nothing", so
  callers never null-check a list.
- **Expose read-only types.** Return `List`/`Map` (not `MutableList`); keep mutation internal.
- **Prefer functional transforms** (`map`/`filter`/`associate`, Java Streams) over manual loops with
  accumulator vars — clearer and less error-prone.
- For **large or chained** transformations, use a `Sequence` (Kotlin) or a `Stream` (Java) to avoid
  building an intermediate collection at every step.
- Don't mutate a collection while iterating it.

## Domain modeling

- **Avoid primitive obsession.** A `DatasetId` value class/record beats passing a raw `String`/`UUID` —
  the type prevents mixing up arguments.

  ```kotlin
  @JvmInline
  value class DatasetId(val value: String)
  ```

  ```java
  public record DatasetId(String value) {}
  ```

- **No anemic models, no god objects.** Put behaviour with the data it operates on; keep cohesion high
  and classes focused.
- Keep each module (Spring Modulith module, or feature package) self-contained: entities and
  repositories stay internal; other modules see only the module's public API — see
  [`formatting-and-static-analysis.md`](./formatting-and-static-analysis.md#package-layout).

### Entity equals, hashCode, and toString

JPA/Hibernate entities are the one place default `data class`/record/Lombok `equals`-`hashCode`
generation causes real bugs:

- **Don't base `equals`/`hashCode` on all fields** (the `data class`/`@Data` default). A lazily-loaded
  proxy or a partially-loaded entity will have `null`/default values for fields not yet fetched,
  breaking equality inconsistently.
- **Base entity identity on a stable natural/business key** if one exists; if you must use the surrogate
  `id`, guard against comparing two *new, unsaved* entities (both have `id == null`, so they'd
  incorrectly compare equal) — most teams exclude `equals`/`hashCode` from `@Data` and hand-write them,
  or use a base class implementing identity-based equality.
- Don't put entities in a `data class` at all — they're mutable, lifecycle-managed objects, not value
  types. Use a plain class with an explicit `equals`/`hashCode`, or Lombok with `equals`/`hashCode`
  excluded and hand-written.
- `toString()` on an entity must not traverse lazy associations — that triggers an extra query (or a
  `LazyInitializationException` outside a session) every time something logs the entity.

## Error handling

- **Fail fast, fail loud.** Validate inputs at the boundary; throw a meaningful domain exception rather
  than returning `null` to signal an error.
- **Never swallow exceptions** — no empty `catch {}`. Catch only what you can handle; otherwise let it
  propagate to the central error handler — see
  [`api-design.md`](./api-design.md#error-handling--rfc-9457-problem-details).
- Catch specific exception types, not `Exception` / `Throwable`, except at a top-level boundary.
- Exception messages state *what and why*, and never contain secrets or PII.

## Persistence & transactions

- **`@Transactional` boundaries live in the service layer** — not controllers, not repositories.
- Keep transactions short — no remote/HTTP calls or message publishing inside an open DB transaction.
- **Beware N+1 queries.** Fetch what you need in one query; don't loop-and-query. Index fields you filter
  or sort on.
- Schema changes go through versioned migrations (Flyway/Liquibase), backwards-compatible
  (expand → migrate → contract).

### `@Transactional(readOnly = true)`

Use it on **service methods that only read** — a query or lookup with no insert/update/delete. Plain
`@Transactional` is for methods that write.

```kotlin
@Transactional(readOnly = true)
fun getAccount(id: AccountId): AccountResponse =
    repository.findById(id)?.toResponse() ?: throw NotFoundException(id)

@Transactional
fun rename(id: AccountId, name: String) {
    val account = repository.findById(id) ?: throw NotFoundException(id)
    account.rename(name)   // writes inside a read-write transaction
}
```

Why it matters on a read path:

- The JPA/Hibernate context skips dirty-checking and the flush before commit — less overhead per query.
- It signals intent and **guards against accidental writes** in a method meant to be read-only.
- It lets the infrastructure route the transaction to a **read replica** where one is configured.

## Pagination

- **Paginate anything that can grow unbounded**; never load a whole collection into memory.
- **Offset pagination** (`LIMIT n OFFSET m`) is fine for small, bounded lists and jump-to-page UIs — but
  it degrades on large data: the database scans and discards all skipped rows, so deep pages get slow,
  and concurrent inserts shift rows between pages.
- **On big data, use keyset (seek) pagination.** Page by the last row's sort key instead of an offset —
  constant time per page and stable under concurrent writes.

  ```sql
  -- first page
  SELECT * FROM event ORDER BY created_at DESC, id DESC LIMIT 50;

  -- next page: seek past the last row seen (composite key, unique tiebreaker)
  SELECT * FROM event
  WHERE (created_at, id) < (:lastCreatedAt, :lastId)
  ORDER BY created_at DESC, id DESC
  LIMIT 50;
  ```

- Keyset requires a **stable, indexed sort key ending in a unique tiebreaker** (e.g. `id`). Return the
  last row's key as the cursor for the next request rather than a page number.
- Use keyset for large tables, infinite scroll, and exports; keep offset only where total count / random
  page access is genuinely needed.

## Date & time

- **Store and transport time in UTC.** Convert to a local zone only at the presentation edge, never in the
  domain or database.
- Use `Instant` / `OffsetDateTime` for points in time; avoid `LocalDateTime` for timestamps (it has no
  zone and silently assumes the server's).
- **Inject a `Clock`** instead of calling `Instant.now()` / `LocalDate.now()` directly — time becomes a
  dependency you can fix in tests.

  ```kotlin
  class SubscriptionService(private val clock: Clock) {
      fun isExpired(sub: Subscription) = sub.expiresAt.isBefore(Instant.now(clock))
  }
  // test: Clock.fixed(...) makes "now" deterministic
  ```

- Compare/serialize consistently (ISO-8601); don't format dates by string concatenation.

## Resource management

- **Close what you open.** Use Kotlin's `use { }` / Java's try-with-resources for streams, files, and any
  `Closeable` — it closes even on exception.

  ```kotlin
  file.inputStream().use { stream -> parse(stream) }
  ```

  ```java
  try (var stream = file.inputStream()) {
      parse(stream);
  }
  ```

- Don't hold connections/streams open across an I/O wait or a long transaction.
- Prefer framework-managed resources (repositories, `RestClient`) over manually opening connections.

## Messaging & concurrency

- **Make consumers idempotent.** Message delivery is at-least-once — reprocessing the same event must be
  safe.
- Handle poison messages via dead-letter; don't infinite-retry.
- Prefer immutable data across threads; avoid shared mutable state.
- **Don't hand-roll unbounded thread pools.** An `Executors.newCachedThreadPool()` under load can exhaust
  memory/file handles; size pools deliberately and prefer the JDK's structured-concurrency /ExecutorService
  patterns (or virtual threads for I/O-bound work — see
  [`spring-boot-practices.md`](./spring-boot-practices.md#virtual-threads)).

## Configuration

- **Externalize config** (env vars / Spring profiles). No hardcoded URLs, ports, or credentials.
- **No secrets in code, logs, or version control** — injected at runtime only.
- Fail on startup if required config is missing, not at first use in production.

## Observability

- **Structured logging** with a correlation ID propagated across service and message boundaries.
- Log at boundaries and on error paths; don't log per-record inside a bulk operation.
- Expose metrics/traces (OpenTelemetry/Micrometer) for operations worth measuring.

## Dependencies

- **No esoteric libraries.** Before adding a dependency: is it maintained, reputable, and not already
  solvable with the stdlib or Spring?
- Reusable code shared across services → publish to a shared artifact repository; don't copy-paste
  between repos.
- Keep a single source of dependency versions (Gradle version catalog / Maven BOM).

## DRY, but not too dry

Remove duplicated logic — but a *little* duplication beats the *wrong* abstraction. Don't couple two
features just because their code looks similar today.

## Code review — the human gate

Standards above are enforced by reviewers, not just CI. A review checks:

- Does it do what the ticket says, with tests proving it?
- Is it readable without the author explaining it?
- Right altitude — no over-engineering, no framework detail leaking into the domain?
- Are error, edge, and null paths handled?

Keep merge/pull requests **small and focused** — one concern per PR reviews faster and regresses less.
