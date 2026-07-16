# Secure Coding

Security is built in from the first line, not reviewed at the end.

**Non-negotiables:**
1. **No secrets in code, logs, or version control.** 2. **Every endpoint has an authorization rule.**
3. **Validate all input at the boundary.** 4. **Never leak stack traces / internal detail to clients.**

## What belongs in a shared security module vs. per-service

If the org has a shared platform library, the security **plumbing is centralized** there — don't
reimplement it per service. The security **decisions are per-service** — a shared library can't know
which role your endpoint needs or which inputs are valid.

| Provided by a shared platform library (reuse) | Owned by each service (you must do) |
|---|---|
| **Token validation** — filter reads `Authorization: Bearer`, validates, populates the security context | **Enable** the shared security module |
| **Authenticated user accessor** — a service to get the current caller's identity/claims | **Annotate every endpoint** with `@PreAuthorize` (which roles), deny-by-default |
| **Method-security infra** — `@PreAuthorize` support + object-level permission expressions | **Call object-level checks** where the resource is scoped (see [Authorization](#authorization)) |
| **Outbound calls** — a shared HTTP client propagating the caller's token + correlation ID | **Validate input** (`@Valid`), avoid injection, don't log secrets |
| **Observability** — correlation ID + OpenTelemetry wiring | **Map** service-specific roles to endpoints; read the caller via the shared accessor |

Rule of thumb: **plumbing (authenticate, expose the user, propagate the token, observe) is shared
infrastructure; policy (who may do what, what input is valid) is the service.**

## Authentication

- Identity provider is an OAuth2/OIDC provider (e.g. Keycloak, Auth0, Okta). Services are **JWT resource
  servers** — validate the bearer token on every request; never trust client-supplied identity fields.
- Configure the JWT decoder from a shared security module, not per service. Realm/tenant, client, issuer
  and JWK-set URI come from **environment variables** — no hardcoded issuers or keys.
- Service-to-service calls authenticate with an injected client secret — never a committed one.

```yaml
platform:
  commons:
    auth:
      enabled: true
```

The required configuration is supplied as **environment variables**, not application config:

```bash
AUTH_USER_SERVICE=http://<auth-service-host>:<port>
AUTH_CLIENT_ID=<client-id>
AUTH_CLIENT_SECRET=<from-secret-store>   # never in application.yml or a committed .properties
```

These come from the deployment layer (Kubernetes/GitOps), not the application repo:

- Non-secret values go in the deployment's config overlay (`ConfigMap` / env) per environment.
- The **secret** is a Kubernetes `Secret` injected into the pod as an env var — never in application
  source or a committed properties file.

**Warning — base64 is not encryption:** a plain `kind: Secret` stores values as **base64 — encoding, not
encryption**. Anyone with read access to the GitOps repo can decode it. Store secret values encrypted at
rest in Git (Sealed Secrets / SOPS / External Secrets), not as raw base64. If a secret is ever committed
in plaintext or pasted into a ticket or chat, **rotate it immediately** and treat it as compromised.

Only add a custom security config when a service needs to **override** the baseline (e.g. permit a
specific public path). Otherwise rely on the shared default (`anyRequest().authenticated()`, JWT resource
server, CSRF off for the stateless API) and express access rules with `@PreAuthorize` on controllers.

## Authorization

**Deny by default.** No public endpoint without a deliberate, documented reason (health probes excepted
and locked down separately).

- Enforce with method security at the controller:

  ```kotlin
  @PreAuthorize("hasAnyRole('curator', 'editor')")
  fun create(@Valid @RequestBody request: CreateResourceRequest): ResourceResponse
  ```

- **Check authorization on the object, not just the type.** "User can read resources" ≠ "user can read
  *this* resource" — this prevents IDOR / broken-object-level-authorization. Use a resource-scoped
  expression:

  ```kotlin
  @PreAuthorize("hasResourcePermission(#resourceId, 'READ')")
  fun get(@PathVariable resourceId: String): ResourceResponse
  ```

- Define roles/permissions in one shared place; don't invent per-service role strings.
- Derive the caller from the security context, **never** from a request param or body — a `userId` in the
  payload is spoofable.

## Input validation

- Validate **all** external input at the boundary with Bean Validation (`@Valid`, `@NotNull`, `@Size`,
  `@Pattern`, …). Reject early with `400`.
- **Allow-list, don't deny-list.** Constrain types, ranges, lengths and formats.
- Treat path params, query params, headers and uploads as untrusted too — not only JSON bodies.
- Validate uploads: content type, size cap, and never derive a filesystem path from a client-supplied
  string (path traversal).

## Injection

- **MongoDB:** use the repository / `Criteria` / `Query` API. Never build a query from string
  concatenation of user input, and never pass user input into `$where` / JS evaluation.
- **SQL:** parameterized queries only. No string-interpolated SQL; bind parameters.
- **No dynamic code / expression evaluation** on user input (SpEL, scripting engines).
- Encode anything reflected back to a client to avoid stored-XSS via the API.

## Secrets & sensitive data

- Secrets (DB creds, client secret, API keys) come from **env / mounted config only** — never source,
  committed defaults, or `.properties` in git.
- **Never log** tokens, passwords, secrets or full PII. Mask or omit; don't log whole request bodies that
  may carry sensitive fields.
- Don't put sensitive data in URLs (query strings hit logs and proxies) — use headers or the body.

## Error & information leakage

- All errors go through the central `@ControllerAdvice` → `ProblemDetail` handler (see
  [`api-design.md`](./api-design.md#error-handling--rfc-9457-problem-details)). Return a generic, safe
  message; log the detail server-side with a correlation ID.
- No stack traces, SQL, class names or file paths in HTTP responses.
- Protect or disable Swagger UI and verbose actuator endpoints in production.

## Transport & headers

- Do not assume a trusted network — validate tokens regardless (zero-trust between services).
- Stateless JWT APIs disable session CSRF, which is correct **only** because there are no cookie-based
  sessions. Don't reintroduce session auth without CSRF protection.
- Configure CORS explicitly (allowed origins); never `*` combined with credentials.

## Dependencies & supply chain

- Run a **dependency/container scanner in CI** (Trivy, Snyk, OWASP Dependency-Check, or equivalent). A
  fixable vulnerability is patched as soon as possible and re-tested.
- Vet new dependencies before adding: maintained, reputable, not already solvable with the stdlib/Spring
  (see [no-esoteric-libraries](./clean-code.md#dependencies)).
- Keep dependencies current; don't pin to a known-vulnerable version to avoid a bump.

## Security testing

- Cover **negative authorization**: assert `401` without a token and `403` for the wrong role — not just
  the happy path.
- Test input validation rejects malformed / oversized / malicious input.
- Test that object-level checks block access to another user's / tenant's resource.

## Quick checklist

- [ ] Endpoint has a `@PreAuthorize` (or a documented public exception)
- [ ] Object-level authorization where the resource is user/tenant-scoped
- [ ] All input `@Valid`-ated; uploads size/type/path-checked
- [ ] No string-built queries; parameterized only
- [ ] No secrets in code/config/logs; caller identity from security context
- [ ] Errors via `ProblemDetail`, no internal detail leaked
- [ ] Negative-auth and validation tests exist
- [ ] Dependency/container scan clean (or fixable findings ticketed)
