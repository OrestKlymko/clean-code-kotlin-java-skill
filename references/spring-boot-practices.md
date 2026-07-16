# Spring Boot Practices

Framework-specific idioms that sit on top of the general clean-code rules in
[`clean-code.md`](./clean-code.md).

## Configuration properties

- Bind config with typed `@ConfigurationProperties`, not scattered `@Value` — one class per concern,
  validated at startup.

  ```kotlin
  @ConfigurationProperties("app.export")
  @Validated
  data class ExportProperties(
      @field:Positive val batchSize: Int,
      @field:NotBlank val outputBucket: String,
  )
  ```

- **`@Validated` + Bean Validation annotations on the properties class** turns a bad config value into a
  startup failure with a clear message, instead of a runtime `NullPointerException` three layers deep.
- Use Spring **profiles** (`application-{profile}.yml`) for environment-specific values; keep
  `application.yml` as the profile-independent defaults.
- Fail on startup if required config is missing — don't let a service start half-configured.

## Transactions

See [`clean-code.md` — Persistence & transactions](./clean-code.md#persistence--transactions) for the
core rule (`@Transactional` in the service layer, `readOnly = true` on read paths).

## Caching

- `@Cacheable`/`@CachePut`/`@CacheEvict` for expensive, repeatable reads — but design the **cache key**
  deliberately (`key = "#id"`, not the default `SimpleKey` over all args when only one matters).
- Always set a **TTL/eviction policy** — an unbounded cache is a memory leak with extra steps.
- Cache invalidation is a correctness concern, not an afterthought: know exactly which write paths must
  evict which keys before adding a cache, not after a stale-data bug.

## Resilience — timeouts, retry, circuit breaking

- **Every outbound HTTP/RPC call has an explicit timeout.** No client call blocks indefinitely; an
  un-timed-out downstream dependency turns into your own outage.
- Use **Resilience4j** for retry, circuit-breaking, rate-limiting, and bulkhead patterns rather than
  hand-rolling retry loops:

  ```kotlin
  @Retry(name = "downstream", fallbackMethod = "fallback")
  @CircuitBreaker(name = "downstream", fallbackMethod = "fallback")
  fun fetchPricing(sku: String): PricingResponse = client.getPricing(sku)

  fun fallback(sku: String, ex: Exception): PricingResponse = PricingResponse.unavailable(sku)
  ```

- **Retries must be idempotent-safe.** Only retry operations that are safe to repeat (GET, or a POST
  guarded by an idempotency key — see [`api-design.md`](./api-design.md#idempotency)).
- A circuit breaker's fallback should degrade gracefully (cached/default value, clear error) — not swallow
  the failure silently.

## HTTP clients

- Prefer Spring's **`RestClient`** (synchronous) or **`WebClient`** (reactive) over the deprecated
  `RestTemplate` for new code.
- Configure **connection timeout, read timeout, and a bounded connection pool** explicitly — the
  framework defaults are not safe assumptions for production traffic.
- Propagate the correlation ID (and, where applicable, the caller's auth token) on every outbound call so
  distributed traces stay connected.

## Database connection pool (HikariCP)

- Size the pool to your actual concurrency needs, not a guess — an oversized pool adds DB-side context-
  switching overhead without added throughput; an undersized one queues requests behind
  `connectionTimeout`.
- Set `connectionTimeout` (how long a thread waits for a connection before failing) and
  `leakDetectionThreshold` (logs a warning if a connection is held longer than expected — catches
  connections that were never closed).
- Match pool size across the service and the database's own `max_connections`, accounting for every
  service instance that connects to the same database.

## Virtual threads

Requires Java 21+ and Spring Boot 3.2+.

- `spring.threads.virtual.enabled=true` puts the request-handling thread pool on virtual threads —
  genuinely useful for **I/O-bound, blocking-style code** (JDBC, blocking HTTP clients, file I/O) with high
  concurrency, since a blocked virtual thread doesn't pin an OS thread.
- **Not a speedup for CPU-bound work**, and virtual threads still **pin the carrier thread inside a
  `synchronized` block** (fixed progressively across JDK versions, but don't rely on it) — prefer
  `ReentrantLock` over `synchronized` in code that runs on virtual threads if this matters to you.
- Don't combine virtual threads with a hand-sized traditional thread pool "just in case" — that defeats
  the point; let the platform thread pool go effectively unbounded and rely on downstream backpressure
  (DB pool size, HTTP client pool size) to limit real concurrency.

## Kotlin coroutines vs. blocking I/O

- Coroutines and Spring MVC's blocking servlet model don't mix well: a `suspend` controller method
  backed by a blocking JDBC call still blocks a thread — you get coroutine *syntax* without a coroutine
  *benefit*.
- Use coroutines for genuine async workloads: Spring **WebFlux** + **R2DBC**, or wrapping a blocking call
  explicitly with `withContext(Dispatchers.IO)` so it's clear the block is deliberate and isolated.
- Don't launch unstructured coroutines (`GlobalScope.launch`) in request-handling code — tie every
  coroutine to a scope that's cancelled when the request/parent completes.

## Actuator & observability

- Expose only the endpoints you need (`management.endpoints.web.exposure.include`); `env`, `heapdump`,
  and `threaddump` are diagnostic gold for an attacker — lock them down or exclude them in production.
- Use `/actuator/health/liveness` and `/actuator/health/readiness` for Kubernetes liveness/readiness
  probes, not a bespoke health endpoint.
- **Micrometer metric names in dot.case** (`orders.created`), and keep **tag cardinality bounded** — a tag
  with a user ID or a UUID per request will blow up your metrics backend's cardinality.
- Structured (JSON) logs in production, with the correlation ID as a consistent field, so log aggregation
  can join a request across services.

## Bean lifecycle

- **No heavy work in constructors.** Constructor injection should just assign fields; defer expensive
  initialization to `@PostConstruct` (and only when it can't just happen lazily on first use).
- **Beans are singletons by default — keep them stateless or thread-safe.** A mutable instance field on a
  `@Service` is shared across every concurrent request; if it holds request-scoped state, that's a
  concurrency bug waiting for load.
- Avoid `@Autowired` setter injection for "optional" dependencies; use a constructor parameter with a
  sensible default, or split into a separate bean.

## Feature flags

For gradual rollout of risky changes, gate behavior behind a feature flag (a config property, or a
dedicated flag service) rather than a long-lived branch — it lets you ship the code dark and turn it on
without a deploy, and roll back the same way.
