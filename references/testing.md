# Testing Standards

Developer-written unit and integration tests living in `src/test/…` inside each service.

## Frameworks

| Concern | Kotlin | Java |
|---|---|---|
| Test runner | **JUnit 5** | **JUnit 5** |
| Mocking | **MockK** | **Mockito** |
| Integration DB | **Testcontainers** | **Testcontainers** |
| Assertions | kotlin-test / AssertJ | AssertJ |
| Module boundaries | Spring Modulith `ApplicationModules` test | ArchUnit (equivalent architecture tests) |

Do not mix Mockito and MockK in the same Kotlin service.

## Unit vs integration

- **Unit tests** — no Spring context. Mock collaborators. Fast; the bulk of the suite.
- **Integration tests** — `@SpringBootTest` and/or **Testcontainers** against a real Postgres/Mongo. Name
  them with an **`*IT`** suffix so they are distinguishable and can run in a separate CI stage.

**Any repository, query, or migration logic must have an integration test against a real container** —
mocking the database proves nothing about the query.

```kotlin
@Testcontainers
@SpringBootTest
class AccountRepositoryIT {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:16")
    }

    @Test
    fun `persists and reloads account`() { /* ... */ }
}
```

For CI speed, enable **Testcontainers reuse** (`testcontainers.reuse.enable=true` in
`~/.testcontainers.properties`, and `.withReuse(true)` on the container) so a local/CI run doesn't pay
container-startup cost per test class — and run independent test classes in **parallel** where the
framework and container strategy allow it.

## Test naming

Pick the idiomatic style per language and be consistent within a service.

**Kotlin — backtick, given/when/then narrative:**

```kotlin
@Test
fun `given existing account when fetched then returns mapped response`() { }

@Test
fun `given missing account when fetched then throws not found`() { }
```

**Java — `@Nested` + `@DisplayName`, method `should_condition`:**

```java
@Nested
@DisplayName("createAccount")
class CreateAccount {

    @Test
    void shouldCreateAndReturnDto_whenValid() { }

    @Test
    void shouldThrowConflict_whenEmailTaken() { }
}
```

Avoid meaningless names (`test1`, `testCreate`). The name states **what behaviour** is verified.

## Structure of a test

Arrange–Act–Assert (given/when/then). One logical assertion per test where practical.

```kotlin
@Test
fun `given valid request when create then account is stored`() {
    // given
    val request = CreateAccountRequest(email = "a@b.example")
    every { repository.save(any()) } returns storedAccount

    // when
    val result = service.create(request)

    // then
    result.id shouldNotBe null
    verify { repository.save(any()) }
}
```

## What to test

- **Do** test business logic: services, calculators, mappers, validators, error paths, edge cases.
- **Do** test the web contract (`@WebMvcTest`): status codes, error shape, validation, authorization.
- **Do** test negative-authorization paths (`401`/`403`) — see [`security.md`](./security.md).
- **Don't** write tests that only assert a mock was called with no real logic between.

## Fixtures & test data

- Mirror the main source package under `src/test/…`.
- Shared fixtures/builders go in a `fixtures`/`support` package or `testFixtures` — don't copy-paste
  object setup across tests.
- Prefer a **test data builder / object mother** (`anAccount().withEmail(...).build()`) over repeating
  large constructor calls — it keeps tests readable and isolates them from constructor signature changes.

## Coverage

- **Tool:** JaCoCo (both Gradle and Maven).
- **Threshold:** a common baseline is **≥ 80% instruction coverage**, enforced — the build **fails**
  below it. Tune the number to your project; treat it as a floor, not a target.
- **Scope:** verified on every merge/pull request and on the main branch.

**Gradle (Kotlin):**

```kotlin
plugins {
    jacoco
}

tasks.withType<Test> {
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                counter = "INSTRUCTION"
                minimum = "0.8".toBigDecimal()
            }
        }
    }
}

tasks.check { dependsOn(tasks.jacocoTestCoverageVerification) }
```

**Maven (Java):**

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>check</id>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>BUNDLE</element>
            <limits>
              <limit>
                <counter>INSTRUCTION</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.80</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

**What to exclude:** application entrypoint / `*Application` main class, generated code (MapStruct impls,
OpenAPI-generated clients/models), plain DTOs/records with no behaviour, `@Configuration` classes that
only wire beans. **Don't** exclude a class just to hit the number — exclusions are for genuinely
untestable boilerplate.

**Guidance:** coverage measures *executed* lines, not *asserted* behaviour. A high number with weak
assertions is worse than an honest lower one.

## No BDD/Cucumber at unit level

Gherkin/Cucumber is not used for developer-written unit tests — the backtick / `@DisplayName` narrative
naming already gives readable specs without the indirection of step definitions. Reserve
Gherkin/Cucumber for a dedicated QA/acceptance automation layer, if the org has one.

## Contract testing (optional, for inter-service APIs)

If multiple services depend on each other's APIs and integration environments are expensive/flaky,
consider **consumer-driven contract testing** (e.g. Pact) so a producer can verify it still satisfies
every consumer's expectations without a full end-to-end environment. Not needed for a single monolith or
a handful of tightly-coordinated services — add it when the number of inter-service contracts makes
manual coordination risky.
