# API Design

How to design, document, and enforce REST APIs. Where these rules are silent, defer to published prior
art rather than inventing a local style.

## Design principles & prior art

Design REST-fully first: resources (nouns), correct verbs, correct status codes, stateless requests,
consistent naming. These are settled industry conventions — when a case isn't covered below, defer to
these guidelines (in priority order):

| Guideline | Use it for | Link |
|---|---|---|
| **Zalando RESTful API Guidelines** | Primary reference — naming, versioning, pagination, errors, compatibility rules | [opensource.zalando.com](https://opensource.zalando.com/restful-api-guidelines/) |
| **Microsoft REST API Guidelines** | Collections, filtering, long-running ops, standard query params | [github.com/microsoft/api-guidelines](https://github.com/microsoft/api-guidelines) |
| **Google API Design Guide** | Resource-oriented design, naming, standard methods | [cloud.google.com/apis/design](https://cloud.google.com/apis/design) |
| **RFC 9457 — Problem Details** | The error body format (see below) | [rfc-editor.org/rfc/rfc9457](https://www.rfc-editor.org/rfc/rfc9457) |

Core rules, non-negotiable:

- **Resource-oriented.** URLs name resources (nouns), not actions. State lives in the resource, not the
  verb.
- **Stateless.** Every request carries its own auth/context; no server-side session between calls.
- **Consistent over clever.** Same concept → same name, same shape, every service. A consumer should
  guess the next endpoint correctly.
- **Backward-compatible by default.** Additive changes only within a version; breaking changes bump the
  version. See Zalando's compatibility section for what counts as breaking.

## URL & versioning

- Base path `/api/v1/<resource>`, plural resource nouns, kebab-case for multi-word segments (e.g.
  `/api/v1/agenda-systems`).
- Version in the path (`v1`). Breaking changes bump the version; additive changes do not.
- Use HTTP verbs correctly: `GET` (read), `POST` (create), `PUT`/`PATCH` (update), `DELETE` (remove).
- Use correct status codes: `200`/`201`/`204`, `400`, `401`, `403`, `404`, `409`, `422`.

## Controllers

- `@RestController`, thin — no business logic. Delegate to a `*Service`.
- Prefer an **interface + `*ControllerImpl`** split so the OpenAPI annotations and contract live on the
  interface.
- Validate input with Bean Validation (`@Valid`, `@NotNull`, …); never trust the client.

```kotlin
interface AccountController {
    @PostMapping("/api/v1/accounts")
    fun create(@Valid @RequestBody request: CreateAccountRequest): AccountResponse
}
```

## DTOs

- Suffix request/response types **`*Request` / `*Response`**. Use `*Dto` only for internal transfer
  objects that are neither.
- In Kotlin, DTOs are `data class`es; in Java, prefer `record`s (see
  [`clean-code.md`](./clean-code.md#java-idioms)). Map to/from domain with extension functions
  (`toResponse()`) or a dedicated `*Mapper`.
- **Never expose persistence entities over the wire** — always map to a DTO.

```kotlin
data class CreateAccountRequest(
    @field:Email val email: String,
    @field:NotBlank val displayName: String,
)

data class AccountResponse(
    val id: String,
    val email: String,
    val displayName: String,
)

fun Account.toResponse() = AccountResponse(id.value, email, displayName)
```

## Error handling — RFC 9457 Problem Details

A single **`@RestControllerAdvice`** per service translates exceptions to `ProblemDetail`
([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457), which obsoletes RFC 7807 and is wire-compatible),
served as `application/problem+json`.

```kotlin
@RestControllerAdvice
class ExceptionTranslator {

    @ExceptionHandler(EntityNotFoundException::class)
    fun handleNotFound(ex: EntityNotFoundException): ProblemDetail =
        ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.message).apply {
            type = URI.create("https://api.example.com/problems/not-found")
            title = "Resource not found"
        }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed").apply {
            title = "Invalid request"
        }
}
```

- One central advice, not scattered `try/catch` in controllers.
- Map domain exceptions → HTTP status consistently (`404`, `409`, `400`, `403`).
- Do not leak stack traces or internal messages to clients; log the detail server-side.

## OpenAPI

- Every REST service exposes an **OpenAPI 3.1** spec via `springdoc` at a stable URL (Swagger UI + JSON).
- **Commit the generated spec** to the repo root (`<service>.yaml`) so consumers can diff the contract in
  code review.
- Annotate the controller interface (`@Operation`, `@ApiResponse`) so the spec is meaningful.
- Non-REST services (WebSocket / AMQP) document their message contracts in `README.md` instead.

## Linting & validation

Lint the generated OpenAPI spec **in CI**, not by eye. A CI gate should fail the build on missing
descriptions/examples/`operationId`, an undeclared security scheme, or a non-3.1 spec.

- **[vacuum](https://quobix.com/vacuum/)** ([`daveshanley/vacuum`](https://github.com/daveshanley/vacuum))
  — a single Go binary, no Node runtime needed, fast, native OpenAPI 3.0/3.1/3.2 support. Runs
  [Spectral](https://github.com/stoplightio/spectral)-format rulesets, so the shared ruleset is portable
  across the ecosystem.

  ```bash
  vacuum lint -r openapi-ruleset.yaml build/openapi.json
  ```

  A good shared ruleset requires: `description` / `examples` / `operationId`, a complete `info` block, a
  declared `securityScheme`, OpenAPI 3.1, and explicit nullability (`@Schema(nullable = true)` — springdoc
  does **not** auto-detect Jakarta `@Nullable`).

- **Breaking-change gate: [oasdiff](https://github.com/oasdiff/oasdiff).** Diffs the new spec against the
  last released tag's committed spec; fails the build on a breaking change unless it's explicitly approved
  and the version is bumped.

  ```bash
  oasdiff breaking baseline-openapi.json build/openapi.json --fail-on ERR
  ```

- Ecosystem alternatives: **Spectral** (the original, Node-based, same ruleset format),
  **[Redocly CLI](https://github.com/Redocly/redocly-cli)** (`redocly lint`), and
  **[Zally](https://github.com/zalando/zally)** (Zalando's linter enforcing their guidelines directly).

## Pagination & filtering

- Paginated list endpoints return Spring `Page`/`Slice` (or a stable custom envelope) with `page`, `size`,
  `totalElements`. Never return an unbounded list.
- Keep filter/sort query-param names consistent across services.
- See [`clean-code.md` — Pagination](./clean-code.md#pagination) for offset vs. keyset pagination.

## Idempotency

- **`POST` endpoints that create a resource should support an idempotency key** so a client retry after a
  timeout doesn't create a duplicate. Accept an `Idempotency-Key` header, store the key with the created
  resource (or the response) for a bounded window, and return the original result on a repeated key
  instead of creating a second resource.
- This is the REST-API counterpart of the message-consumer idempotency rule in
  [`clean-code.md`](./clean-code.md#messaging--concurrency) — the same "at-least-once delivery" problem
  shows up whenever a client can legitimately retry a write.
- `PUT` is idempotent by definition (same request, same result) — this concern is specifically for `POST`.
