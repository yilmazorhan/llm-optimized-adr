# Issue: Integration Testing Strategy

Unit tests (ADR-32) verify domain logic with mocked ports, but adapters, REST endpoints, Flyway migrations, and security filters only behave as in production when exercised against real infrastructure. Without a standard, integration tests boot inconsistent environments, depend on developer-local services, validate against a database dialect that differs from production, and run in the same phase as unit tests.

**Impact**: Migrations fail only in production, adapter bugs surface at deployment, REST error contracts (ADR-23) go unverified, and CI times become unpredictable.

# Decision:

Adopt `@QuarkusTest` with a **two-tier infrastructure strategy** mapped to the multi-module structure (ADR-09): HSQLDB (Oracle compatibility mode, BR-001) for fast adapter tests, and Testcontainers-backed Oracle (via Quarkus Dev Services, ADR-07) for production-parity tests. REST endpoints **MUST** be tested with REST Assured; integration tests **MUST** run in the Failsafe `verify` phase, separated from Surefire unit tests.

**Required Components**:
- **Tier 1 — HSQLDB (fast)**: Persistence/adapter modules **MUST** use in-memory HSQLDB in Oracle compatibility mode (`sql.syntax_ora=true`, BR-001) via the `%test` profile (ADR-15) with Flyway migrations applied at start (ADR-14)
- **Tier 2 — Oracle parity**: The `container-app` module **MUST** run `@QuarkusTest` against a disposable Oracle database started by Quarkus Dev Services (Testcontainers-backed, ADR-07), matching the production database kind (BR-001)
- **REST Assured**: HTTP endpoint integration tests **MUST** use REST Assured with given/when/then style
- **Phase Separation**: Integration tests (named `*IntegrationTest`) **MUST** execute via the Maven Failsafe plugin in the `verify` phase; unit tests via Surefire in the `test` phase
- **Tagging**: Every `@QuarkusTest` **MUST** carry `@Tag("integration")` (ADR-32 classification)
- **Test Profiles**: Environment variations **MUST** use `QuarkusTestProfile` implementations, not profile-specific test source trees
- **Security**: Secured endpoints **MUST** be tested with `@TestSecurity` tokens (ADR-18); security **MUST NOT** be disabled for integration tests
- **Error Contract**: Error-path tests **MUST** assert `application/problem+json` responses with ADR-17 error codes (ADR-23)
- **No Skips**: Integration tests **MUST** run in `./mvnw clean verify` (ADR-12)

# Ownership

| Role | Person | Competencies |
|------|--------|-------------|
| **Responsible** | Senior Backend Developer | `@QuarkusTest` and `QuarkusTestProfile` authoring, HSQLDB Oracle-compatibility `%test` datasource configuration (BR-001), Quarkus Dev Services / Testcontainers Oracle setup, Flyway test migration lifecycle (ADR-14), REST Assured endpoint assertions including RFC 9457 `problem+json` verification (ADR-23), `@TestSecurity` authenticated testing (ADR-18), Surefire/Failsafe phase binding |
| **Approver** | Tech Lead / Software Architect | Integration scope policy (what must be integration-tested vs. unit-tested), production-parity tiering decisions (HSQLDB vs. Testcontainers), CI container resource and time budget governance, test environment strategy alignment with the ADR-07 graduated approach |

# Constraints:

- Integration tests **MUST NOT** depend on manually started, developer-local services (Docker Compose stacks, running databases); all dependencies **MUST** start automatically within the test run
- Integration tests **MUST NOT** run in parallel (shared database state); parallel execution is reserved for unit tests (ADR-32)
- Each integration test class **MUST** start from a clean database state (Flyway re-migration or transactional rollback)
- Tier 2 tests **MUST** execute against Oracle, matching the production database kind (BR-001); HSQLDB **MUST NOT** be the only validation target for persistence adapters
- `Thread.sleep()` **MUST NOT** appear in any test (ADR-32); readiness polling **MUST** use Awaitility or Quarkus-managed startup
- REST assertions on error paths **MUST** verify `Content-Type: application/problem+json` and the `errorCode` extension field (ADR-23)
- `%test.quarkus.hibernate-orm.database.generation` **MUST** be `none`; schema changes in tests **MUST** come from Flyway scripts only (ADR-14, ADR-15)
- The `container-app` integration suite **SHOULD** complete within 5 minutes on CI hardware including container startup
- Tests **MUST NOT** be skipped or disabled without a tracking identifier in an adjacent comment (ADR-12)

# Alternatives:

1. **HSQLDB-Only Integration Testing (no production-parity tier)**: Run all adapter and endpoint tests against in-memory HSQLDB, skipping Oracle entirely. Rejected because: Flyway's supported-databases documentation classifies SQL migrations as database-specific — scripts written for one database are not guaranteed to run on another (https://documentation.red-gate.com/flyway/docs/getting-started/supported-databases). Oracle-specific DDL (e.g., `NUMBER` precision semantics, `VARCHAR2` semantics, `SEQUENCE ... START WITH ... INCREMENT BY`, `CONNECT BY` hierarchical queries) fails or behaves differently on HSQLDB even in Oracle compatibility mode, so migration defects would only surface in production. Tier 1 remains valuable for speed (BR-001), but it cannot substitute for production-parity validation.

2. **Shared Docker Compose Test Environment**: Start Jaeger/Oracle/etc. via `docker-compose up` before the test run, point tests at fixed hostnames and ports. Rejected because: service lifecycle is external to the build — nothing guarantees the stack is running, current, or healthy when tests execute, and fixed ports conflict across parallel CI jobs. Quarkus Dev Services starts and stops disposable containers automatically per test run via Testcontainers, with container cleanup handled by the library (https://quarkus.io/guides/databases-dev-services). This is the same graduated conclusion as ADR-07: Compose serves long-running development, not automated tests.

# Rationale:

`@QuarkusTest` boots the complete application context — CDI beans, configuration, RESTEasy Reactive endpoints, security filters — validating wiring that unit tests with mocked ports cannot reach (https://quarkus.io/guides/getting-started-testing). Quarkus Dev Services detects the configured database kind and launches a matching disposable container via Testcontainers when no JDBC URL is set, removing manual orchestration from the test lifecycle (https://quarkus.io/guides/databases-dev-services). The two-tier split follows the module boundaries of ADR-09: persistence modules (External System type) validate adapters quickly on HSQLDB in Oracle compatibility mode (BR-001), while the `container-app` deployment assembly — the module whose `db-kind` is the production Oracle — runs the parity tier. `db-kind` is a build-time property in Quarkus, so the two tiers **MUST** live in separate modules rather than test profiles of one module (https://quarkus.io/guides/databases-dev-services).

The Maven Failsafe plugin executes `*IntegrationTest` classes in the `integration-test` phase and verifies outcomes in `verify`, by design separating slow integration cycles from the Surefire `test` phase while keeping both inside `./mvnw clean verify` (https://maven.apache.org/surefire/maven-failsafe-plugin/). REST Assured provides given/when/then HTTP assertions with JSON path matching on response bodies, used by the official Quarkus testing guide for endpoint validation (https://rest-assured.io/). `@TestSecurity` injects test JWT security identities without real key infrastructure, exercising the actual security filters of ADR-18 (https://quarkus.io/guides/security-testing). Awaitility provides declarative readiness polling as the substitute for forbidden `Thread.sleep()` calls (https://github.com/awaitility/awaitility).

# Implementation Guidelines:

## 1. Maven Dependencies (parent `pom.xml`)

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-junit5</artifactId>
    <scope>test</scope> <!-- version from Quarkus BOM (ADR-08) -->
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope> <!-- version managed by Quarkus BOM -->
</dependency>
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-test-security</artifactId>
    <scope>test</scope> <!-- @TestSecurity (ADR-18) -->
</dependency>
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <version>2.7.3</version>
    <scope>test</scope> <!-- Tier 1 (BR-001) -->
</dependency>
```

> **Note**: Versions are managed in the parent `pom.xml` `<dependencyManagement>` / BOM import (ADR-03, ADR-08). Update versions there and verify with `./mvnw clean verify`.

## 2. Surefire / Failsafe Phase Split (parent `pom.xml`)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
    </configuration>
</plugin>
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.5.2</version>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Both phases run inside `./mvnw clean verify` — no test category is skipped (ADR-12). `-Dgroups=integration` selects integration tests only for targeted runs.

## 3. Tier 1 Configuration — HSQLDB Oracle Compatibility (persistence modules, `%test`)

```properties
# %test profile (ADR-15) — in-memory HSQLDB, Oracle compatibility mode (BR-001)
%test.quarkus.datasource.db-kind=hsqldb
%test.quarkus.datasource.jdbc.url=jdbc:hsqldb:mem:testdb;DB_CLOSE_DELAY=-1;sql.syntax_ora=true
%test.quarkus.hibernate-orm.database.generation=none
%test.quarkus.flyway.migrate-at-start=true
%test.quarkus.flyway.clean-at-start=true
```

`sql.syntax_ora=true` enables Oracle SQL syntax per BR-001. `clean-at-start=true` + `migrate-at-start=true` guarantee a clean, fully migrated schema per test class (ADR-14).

## 4. Tier 2 Configuration — Oracle Parity (`container-app`)

```properties
# container-app application.properties (build-time db-kind = production database)
quarkus.datasource.db-kind=oracle
# No %test JDBC URL: Dev Services starts a disposable Oracle container
# via Testcontainers per test run (ADR-07) and injects the URL automatically.
%test.quarkus.hibernate-orm.database.generation=none
%test.quarkus.flyway.migrate-at-start=true
```

The same Flyway scripts that run in production (ADR-14) are the only schema source — verifying them against the production dialect is the purpose of this tier.

## 5. Adapter Integration Test Example (Tier 1)

```java
@QuarkusTest
@Tag("integration")
class JdbcUserRepositoryAdapterIntegrationTest {

    @Inject
    UserRepositoryAdapter adapter; // implements UserRepositoryPort (ADR-10, ADR-11)

    @Test
    void shouldPersistAndFindByEmail() {
        User user = TestUsers.john();

        User persisted = adapter.persistAndFlush(user)
            .await().atMost(Duration.ofSeconds(5));

        Optional<User> found = adapter.findByEmail(user.getEmail())
            .await().atMost(Duration.ofSeconds(5));

        assertThat(found).isPresent();
        assertThat(found.get().getId()).isEqualTo(persisted.getId());
    }
}
```

## 6. REST Resource Integration Test Example (Tier 2, REST Assured)

```java
@QuarkusTest
@Tag("integration")
@TestSecurity(user = "admin", roles = "ADMIN") // exercises real security filters (ADR-18)
class UserResourceIntegrationTest {

    @Test
    void shouldCreateUserAndReturn201() {
        given()
            .contentType(ContentType.JSON)
            .body(TestUsers.createJohnRequest()) // DTO from ADR-24
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(201)
            .body("email", equalTo("john@example.com"));
    }

    @Test
    void shouldReturnProblemJsonWithErrorCodeOnDuplicateEmail() {
        given()
            .contentType(ContentType.JSON)
            .body(TestUsers.createJohnRequest()) // user already persisted by @BeforeEach
        .when()
            .post("/api/v1/users")
        .then()
            .statusCode(409)
            .contentType("application/problem+json") // RFC 9457 (ADR-23)
            .body("errorCode", equalTo("DOM-001"))   // ADR-17 error code registry
            .body("correlationId", notNullValue());  // ADR-16 correlation
    }
}
```

## 7. Naming Conventions

| Artifact | Convention | Example |
|----------|-----------|---------|
| Integration test class | `[ClassUnderTest]IntegrationTest` + `@Tag("integration")` (ADR-32) | `UserResourceIntegrationTest` |
| Test profile | `[Purpose]TestProfile` | `PostgresParityTestProfile` |
| Test data builders | Reuse ADR-32 factories | `TestUsers.john()` |
| REST request bodies | Reuse ADR-24 request DTOs | `TestUsers.createJohnRequest()` |

## 8. Maven Commands

| Command | Purpose | Notes |
|---------|---------|-------|
| `./mvnw clean verify -pl container-app -am` | Full build: unit + integration + coverage gate | `-am` per ADR-09; no skips (ADR-12) |
| `./mvnw verify -Dgroups=integration` | Integration tests only | Tag-filtered targeted run |
| `./mvnw verify -DexcludedGroups=integration` | Unit tests only (local fast loop) | `clean verify` still runs all |
| `./mvnw failsafe:verify -pl <module>` | Check IT results only | After `integration-test` phase |

# Additional Recommendations:

- **RECOMMENDED** Keep Tier 1 suites lean (schema-only interactions); anything exercising Oracle-specific SQL belongs in Tier 2.
- **RECOMMENDED** Assert health endpoint responses (`/q/health`) in one Tier 2 test per application to verify the ADR-20 observability wiring.
- **RECOMMENDED** Use `@QuarkusTestProfile` for scenarios requiring different configuration (feature flags, alternative mocks) rather than duplicating test classes.
- **RECOMMENDED** Merge JaCoCo execution data from Surefire and Failsafe so the ADR-32 coverage gate sees combined coverage.
- **MAY** add a Testcontainers-based OTLP collector container when trace export needs verification; tracing assertions **SHOULD** stay optional to keep suite time within budget.
- **RECOMMENDED** Review `target/failsafe-reports/` in CI artifacts on failures — container startup logs appear there.
- **MUST** Reference ADR-07 (Containerized Development Environment), ADR-08 (Quarkus Framework), ADR-09 (Multi-Module Structure), ADR-12 (Code Quality Standards), ADR-14 (Data Migration Strategy), ADR-15 (Application Configuration Management), ADR-16 (Structured Logging), ADR-17 (Exception Handling), ADR-18 (Security Implementation), ADR-23 (REST API Error Handling), ADR-24 (Data Transfer & Validation), ADR-32 (Unit Testing Strategy), BR-001 (HSQLDB Test Infrastructure).
- See Quarkus Testing Guide: https://quarkus.io/guides/getting-started-testing
- See Quarkus Dev Services: https://quarkus.io/guides/databases-dev-services
- See Maven Failsafe Plugin: https://maven.apache.org/surefire/maven-failsafe-plugin/
- See REST Assured: https://rest-assured.io/
- See Testcontainers: https://java.testcontainers.org/
- See Awaitility: https://github.com/awaitility/awaitility
- See Flyway Supported Databases: https://documentation.red-gate.com/flyway/docs/getting-started/supported-databases

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| REST Assured Dependency | `grep rest-assured pom.xml` | Dependency present (test scope) |
| HSQLDB Test Dependency | `grep hsqldb pom.xml` | Dependency present (test scope) |
| Failsafe Configured | `grep failsafe pom.xml` | Plugin present with `integration-test` + `verify` goals |
| Surefire Exclusion | `grep -A2 surefire pom.xml` | `*IntegrationTest` excluded from `test` phase |
| All ITs Tagged | `grep -L '@Tag("integration")' $(find . -name "*IntegrationTest.java")` | Zero output |
| All ITs Use QuarkusTest | `grep -L '@QuarkusTest' $(find . -name "*IntegrationTest.java")` | Zero output |
| HSQLDB %test Profile | `grep '%test.quarkus.datasource' */src/main/resources` | HSQLDB URL present in persistence modules |
| Hibernate DDL Disabled | `grep '%test.*hibernate-orm.database.generation=none'` | Present in all test configs (ADR-14) |
| Flyway in Tests | `grep '%test.quarkus.flyway.migrate-at-start=true'` | Present |
| Error Contract Tested | `grep -r "problem+json" */src/test/java` | At least one match (ADR-23) |
| No Thread.sleep | `grep -r "Thread.sleep" */src/test/java` | Zero matches (ADR-32) |
| Full Verify Passes | `./mvnw clean verify -pl container-app -am` | Exit code 0, no skipped tests (ADR-12) |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Integration Testing Strategy..."
FAIL=0

# Check integration test dependencies
for dep in rest-assured hsqldb; do
  if grep -rq "<artifactId>$dep</artifactId>" pom.xml */pom.xml 2>/dev/null; then
    echo "✅ $dep dependency present"
  else
    echo "❌ $dep dependency not found"
    FAIL=1
  fi
done

# Check Failsafe plugin with both goals
if grep -rq "maven-failsafe-plugin" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ Failsafe plugin configured"
else
  echo "❌ Failsafe plugin not found"
  FAIL=1
fi

# Check all IntegrationTest classes are tagged and use @QuarkusTest
UNTAgged=0
for f in $(find . -name "*IntegrationTest.java" 2>/dev/null); do
  grep -q '@Tag("integration")' "$f" || { echo "❌ Missing @Tag in $f"; UNTAgged=1; }
  grep -q '@QuarkusTest' "$f" || { echo "❌ Missing @QuarkusTest in $f"; UNTAgged=1; }
done
if [ "$UNTAgged" -eq 0 ]; then
  echo "✅ All IntegrationTest classes tagged with @Tag(\"integration\") and @QuarkusTest"
fi

# Check %test HSQLDB configuration (Tier 1, BR-001)
if grep -rq '%test.quarkus.datasource.jdbc.url=jdbc:hsqldb' */src/main/resources 2>/dev/null; then
  echo "✅ HSQLDB %test profile configured (Tier 1)"
else
  echo "❌ HSQLDB %test datasource not configured"
  FAIL=1
fi

# Check Hibernate DDL disabled in test profiles (ADR-14, ADR-15)
if grep -rq '%test.quarkus.hibernate-orm.database.generation=none' */src/main/resources 2>/dev/null; then
  echo "✅ Test Hibernate DDL generation disabled"
else
  echo "❌ %test Hibernate DDL not set to none"
  FAIL=1
fi

# Check RFC 9457 error contract assertions (ADR-23)
if grep -rq 'problem+json' */src/test/java 2>/dev/null; then
  echo "✅ problem+json error contract asserted in tests"
else
  echo "❌ No application/problem+json assertions found"
  FAIL=1
fi

# No Thread.sleep in tests (ADR-32)
SLEEP_COUNT=$(grep -rn "Thread\.sleep" */src/test/java 2>/dev/null | wc -l | tr -d ' ')
if [ "$SLEEP_COUNT" -eq 0 ]; then
  echo "✅ No Thread.sleep in tests"
else
  echo "❌ Thread.sleep found in tests: $SLEEP_COUNT occurrences"
  FAIL=1
fi

# Full verification — includes Failsafe integration tests (ADR-12: no skips)
./mvnw clean verify -pl container-app -am -q \
  && echo "✅ Full build with integration tests passed" \
  || { echo "❌ Build failed"; FAIL=1; }

if [ "$FAIL" -ne 0 ]; then
  echo "❌ Integration Testing Strategy validation FAILED"
  exit 1
fi
echo "✅ All Integration Testing Strategy criteria met"
```
