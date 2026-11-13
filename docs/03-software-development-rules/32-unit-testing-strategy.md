# Issue: Unit Testing Strategy

Hexagonal Quarkus microservices (ADR-08, ADR-10) require standardized unit testing with a clear boundary between unit and integration tests. Without rules, tests boot the full Quarkus context for pure domain logic, execute slowly, and coverage is not enforced.

**Impact**: Slow feedback loops, untestable domain logic, unverifiable coverage, inconsistent test structure across modules.

# Decision:

Adopt JUnit 5 (Jupiter) with Mockito and AssertJ as the unit testing stack, with hexagonal test classification and JaCoCo coverage gates.

**Required Components**:
- **JUnit 5 Platform**: All tests **MUST** run on the JUnit 5 Jupiter platform (`quarkus-junit5`)
- **Hexagonal Test Classification**: Domain layer **MUST** be tested as pure unit tests without Quarkus context; `@QuarkusTest` **MUST** be tagged `@Tag("integration")` and classified as integration test
- **Port Mocking**: Mockito **MUST** be used to mock port interfaces (ADR-10, ADR-11); infrastructure internals **MUST NOT** be mocked in domain tests
- **Assertions**: AssertJ **MUST** be used for value assertions; reactive `Uni`/`Multi` results **MUST** be resolved with `await().atMost(...)` before assertion
- **Structure**: Test classes **MUST** mirror production packages (ADR-11) and be named `[ClassUnderTest]Test`
- **Coverage Gate**: JaCoCo **MUST** enforce ≥80% line coverage on domain packages via the `check` goal bound to `verify`
- **Isolation**: Unit tests **MUST NOT** require external services or containers (database, Jaeger, Prometheus)
- **No Skips**: Tests **MUST NOT** be skipped in `./mvnw clean verify` (ADR-12)

# Ownership

| Role | Person | Competencies |
|------|--------|-------------|
| **Responsible** | Senior Backend Developer | JUnit 5 / Mockito / AssertJ test authoring, hexagonal test classification (pure domain tests vs. `@QuarkusTest` integration), Mutiny reactive assertion patterns (`Uni.await().atMost`), port interface mocking and in-memory test adapters, JaCoCo coverage gate configuration |
| **Approver** | Tech Lead / Software Architect | Test framework selection trade-offs (JUnit 5 vs. JUnit 4 vs. Spock), unit vs. integration test boundary policy, coverage threshold decisions, test suite time budget governance |

# Constraints:

- Unit tests **MUST** execute without network access, Docker containers, or Quarkus context boot
- Test source code **MUST** follow Google Java Style (ADR-12); Spotless **MUST** pass on `src/test/java`
- `Thread.sleep()` **MUST NOT** appear in any test; asynchronous results **MUST** use `await().atMost(Duration)`
- Tests **MUST NOT** be disabled (`@Disabled`) without a tracking identifier in an adjacent comment
- JaCoCo `check` **MUST** fail the build when domain package line coverage is below 80%
- The unit test phase per module **SHOULD** complete in under 60 seconds on CI hardware
- Error path assertions **MUST** verify ADR-17/ADR-25 error codes (e.g., `USER-BUSINESS-002`), not only exception types
- Mocks in domain tests **MUST** target port interfaces only (ADR-10, ADR-11)

# Alternatives:

1. **JUnit 4 (via Vintage Engine)**: Rejected because: JUnit 4 tests require the `junit-vintage-engine` dependency to execute on the JUnit Platform (https://junit.org/junit5/docs/current/user-guide/). The build already executes ArchUnit tests via `archunit-junit5-api` 1.4.1 on the Jupiter platform (ADR-10), and `quarkus-junit5` is Jupiter-based (https://quarkus.io/guides/getting-started-testing) — JUnit 4 would add a second test engine and API generation to the build without new capability. The JUnit project directs new development and migration to JUnit 5 (https://junit.org/junit5/docs/current/user-guide/).

2. **Spock Framework (Groovy)**: Rejected because: Spock specifications are written in Groovy, adding the Groovy compiler and runtime to a Java 21 build (https://spockframework.org/). `google-java-format` — the formatter Spotless enforces per ADR-12 — processes Java source only (https://github.com/google/google-java-format), so Groovy test sources fall outside the ADR-12 automated formatting pipeline and would require a second toolchain configuration.

# Rationale:

Quarkus `@QuarkusTest` boots the full application context (CDI beans, configuration, extensions) per test run (https://quarkus.io/guides/getting-started-testing). The domain layer in this architecture has zero framework dependencies (ADR-10: domain **MUST NOT** depend on adapter or infrastructure packages), so plain JUnit 5 tests execute domain logic without context startup cost. Ports are Java interfaces (ADR-11), which Mockito can substitute directly or be replaced with hand-written in-memory adapters — the substitution mechanism the hexagonal pattern provides (https://site.mockito.org/). The JUnit 5 platform is already present in the build through ArchUnit (`archunit-junit5-api` 1.4.1, ADR-10), so this decision introduces no second test platform. Mockito 5.x uses the inline mock maker by default and requires Java 11+, covering JDK 21 (ADR-02) (https://site.mockito.org/). JaCoCo instruments bytecode and provides the `check` goal, which fails the build below configured ratio thresholds (https://www.jacoco.org/jacoco/). AssertJ provides typed assertions whose failure messages list expected and actual values (https://assertj.github.io/doc/).

# Implementation Guidelines:

## 1. Maven Dependencies (parent `pom.xml`)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5</artifactId>
    <scope>test</scope> <!-- version from Quarkus BOM (ADR-08) -->
</dependency>
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.14.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>3.26.3</version>
    <scope>test</scope>
</dependency>
```

> **Note**: Library versions are managed in the parent `pom.xml` (ADR-03). Update versions there and verify with `./mvnw clean verify`.

## 2. Test Classification (Hexagonal — ADR-10)

| Test Type | Scope | Mechanism | External Services |
|-----------|-------|-----------|-------------------|
| **Unit** | Domain and application services | Plain JUnit 5 + Mockito mocks of ports | None |
| **Integration** | Adapters, REST endpoints, persistence | `@QuarkusTest` + `@Tag("integration")` | Testcontainers / HSQLDB (ADR-07, BR-001) |
| **Architecture** | Layer and package rules | ArchUnit (ADR-10, ADR-11) | None |

Mocking CDI beans inside `@QuarkusTest` uses `@InjectMock` from `quarkus-junit5-mockito` (https://quarkus.io/guides/getting-started-testing). `@InjectMock` **MUST NOT** appear in domain unit tests.

## 3. Domain Unit Test Example

```java
package com.copilot.quarkus.domain.user.service; // mirrors production package (ADR-11)

class UserServiceTest {

    private final UserRepositoryPort repository = Mockito.mock(UserRepositoryPort.class);
    private final UserService userService = new UserService(repository);

    @Test
    void shouldCreateUserWhenEmailIsUnique() {
        Mockito.when(repository.findByEmail("john@example.com"))
            .thenReturn(Uni.createFrom().item(Optional.empty()));
        Mockito.when(repository.persistAndFlush(Mockito.any(User.class)))
            .thenAnswer(inv -> Uni.createFrom().item(inv.getArgument(0)));

        User created = userService.createUser(TestUsers.createJohnCommand())
            .await().atMost(Duration.ofSeconds(1));

        assertThat(created.getEmail()).isEqualTo("john@example.com");
    }

    @Test
    void shouldThrowDuplicateEmailErrorWhenEmailAlreadyExists() {
        Mockito.when(repository.findByEmail("john@example.com"))
            .thenReturn(Uni.createFrom().item(Optional.of(TestUsers.john())));

        UserEmailAlreadyExistsException exception = Assertions.assertThrows(
            UserEmailAlreadyExistsException.class,
            () -> userService.createUser(TestUsers.createJohnCommand())
                              .await().atMost(Duration.ofSeconds(1)));

        // ADR-17/ADR-25: error code MUST be asserted, not only the exception type
        assertThat(exception.getErrorContext().getErrorCode()).isEqualTo("USER-BUSINESS-002");
    }
}
```

## 4. Reactive Assertion Rules

| Pattern | Status |
|---------|--------|
| `uni.await().atMost(Duration.ofSeconds(1))` | **MUST** use for `Uni` results |
| `multi.collect().asList().await().atMost(...)` | **MUST** use for `Multi` results |
| `Thread.sleep(...)` | **FORBIDDEN** |
| Subscribing without a terminal operation | **FORBIDDEN** (assertions may run before emission) |

## 5. Coverage Gate (JaCoCo)

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.12</version>
    <executions>
        <execution>
            <id>prepare</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <includes>
                            <include>com.copilot.quarkus.domain.*</include>
                        </includes>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
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

## 6. Naming Conventions

| Artifact | Convention | Example |
|----------|-----------|---------|
| Unit test class | `[ClassUnderTest]Test` | `UserServiceTest` |
| Integration test class | `[ClassUnderTest]IntegrationTest` + `@Tag("integration")` | `UserResourceIntegrationTest` |
| Test method | `should[Expected]When[Condition]` | `shouldThrowDuplicateEmailErrorWhenEmailAlreadyExists` |
| Test data builder | `Test[PluralEntity]` factory | `TestUsers.john()` |
| In-memory adapter | `InMemory[Entity]RepositoryAdapter` (ADR-11 suffix) | `InMemoryUserRepositoryAdapter` |

## 7. Maven Commands

| Command | Purpose | Notes |
|---------|---------|-------|
| `./mvnw test -pl <module> -am` | Run module tests + dependencies | `-am` per ADR-09 |
| `./mvnw test -Dgroups=unit` | Unit tests only | Tag-filtered |
| `./mvnw clean verify` | Full build + coverage gate | Tests **MUST NOT** be skipped (ADR-12) |
| `./mvnw jacoco:report -pl <module>` | Coverage HTML report | `target/site/jacoco/` |

# Additional Recommendations:

- **RECOMMENDED** Use test data builders (`TestUsers.john()`) over inline object construction in every test method.
- **RECOMMENDED** Verify one behavior per test method; the method name **MUST** describe the behavior under test.
- **RECOMMENDED** Use in-memory port adapters for stateful domain behavior; use Mockito mocks for stateless calls.
- **MAY** enable JUnit 5 parallel execution for pure domain tests via `junit-platform.properties` (`junit.jupiter.execution.parallel.enabled=true`); integration tests **MUST NOT** run in parallel.
- **RECOMMENDED** Review the JaCoCo report when adding domain exceptions so every ADR-17 error code has at least one covering test.
- **MUST** Reference ADR-08 (Quarkus Framework), ADR-09 (Multi-Module Structure), ADR-10 (Hexagonal Architecture), ADR-11 (Package Structure), ADR-12 (Code Quality Standards), ADR-17 (Exception Handling), ADR-24 (Data Transfer & Validation).
- See JUnit 5 User Guide: https://junit.org/junit5/docs/current/user-guide/
- See Quarkus Testing Guide: https://quarkus.io/guides/getting-started-testing
- See Mockito: https://site.mockito.org/
- See AssertJ: https://assertj.github.io/doc/
- See JaCoCo: https://www.jacoco.org/jacoco/
- See Mutiny: https://smallrye.io/smallrye-mutiny/latest/

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| JUnit 5 Platform | `grep quarkus-junit5 pom.xml` | Dependency present |
| AssertJ + Mockito Pinned | `grep -E "assertj-core\|mockito-core" pom.xml` | Exact versions declared |
| Domain Tests Context-Free | `grep -rl "@QuarkusTest" */src/test/java \| grep domain` | Zero matches |
| Mirrored Packages | Code review | Test packages mirror production packages (ADR-11) |
| Naming Convention | `find . -path "*src/test/java*" -name "*.java"` | All test classes end with `Test` |
| No Thread.sleep | `grep -r "Thread.sleep" */src/test/java` | Zero matches |
| No Untracked Disables | `grep -r "@Disabled" */src/test/java` | Zero matches without tracking ID |
| Coverage Gate | `./mvnw clean verify` | Exit code 0 (JaCoCo check passes) |
| Tests Not Skipped | `./mvnw clean verify` | Exit code 0 without `-DskipTests` (ADR-12) |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Unit Testing Strategy..."
FAIL=0

# Check JUnit 5 platform
if grep -rq "quarkus-junit5" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ JUnit 5 platform configured"
else
  echo "❌ quarkus-junit5 not found in pom.xml"
  FAIL=1
fi

# Check pinned test libraries
for lib in assertj-core mockito-core; do
  if grep -rq "<artifactId>$lib</artifactId>" pom.xml */pom.xml 2>/dev/null; then
    echo "✅ $lib declared"
  else
    echo "❌ $lib not found"
    FAIL=1
  fi
done

# Domain tests must not boot Quarkus context
DOMAIN_CTX=$(grep -rl "@QuarkusTest" */src/test/java 2>/dev/null | grep "domain" | wc -l | tr -d ' ')
if [ "$DOMAIN_CTX" -eq 0 ]; then
  echo "✅ No @QuarkusTest in domain packages"
else
  echo "❌ @QuarkusTest found in domain packages ($DOMAIN_CTX files)"
  FAIL=1
fi

# No Thread.sleep in tests
SLEEP_COUNT=$(grep -rn "Thread\.sleep" */src/test/java 2>/dev/null | wc -l | tr -d ' ')
if [ "$SLEEP_COUNT" -eq 0 ]; then
  echo "✅ No Thread.sleep in tests"
else
  echo "❌ Thread.sleep found in tests: $SLEEP_COUNT occurrences"
  FAIL=1
fi

# No untracked disabled tests
DISABLED=$(grep -rn "@Disabled" */src/test/java 2>/dev/null | wc -l | tr -d ' ')
if [ "$DISABLED" -eq 0 ]; then
  echo "✅ No disabled tests"
else
  echo "❌ @Disabled found: $DISABLED occurrences (require tracking IDs)"
  FAIL=1
fi

# Coverage gate + full verification (tests MUST NOT be skipped per ADR-12)
./mvnw clean verify -q && echo "✅ Full build with coverage gate passed" || { echo "❌ Build failed"; FAIL=1; }

if [ "$FAIL" -ne 0 ]; then
  echo "❌ Unit Testing Strategy validation FAILED"
  exit 1
fi
echo "✅ All Unit Testing Strategy criteria met"
```
