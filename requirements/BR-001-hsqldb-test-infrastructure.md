# BR-001: HSQLDB Test Infrastructure

## 1. Summary

As a **backend developer**, I want **an in-memory HSQLDB database configured with Oracle compatibility mode, pre-loaded sample table schemas, and realistic test data**, so that **I can validate every query function, type mapping, pagination logic, and masking pipeline locally without requiring a real database instance or Docker containers**.

This requirement establishes the test database layer that enables rapid, isolated validation of all service functions defined in BR-002, BR-003, and BR-004. HSQLDB running in Oracle syntax compatibility mode provides a zero-dependency, JVM-embedded database that supports the same SQL dialect patterns used in production Oracle environments. Sample schemas mirror the production dashboard query targets, and seed data covers all column types, filter combinations, edge cases, and pagination boundaries.

## 2. Business Rules

- **BR-001-01**: The test profile **MUST** use HSQLDB (exact version managed in the parent POM, ADR-03) as the in-memory database engine with Oracle compatibility mode enabled (`sql.syntax_ora=true`). HSQLDB is Tier 1 of the ADR-33 §Decision two-tier strategy and **MUST NOT** be the sole validation of Oracle-specific behavior.
- **BR-001-02**: HSQLDB **MUST** be configured as a Quarkus datasource in the `%test` profile using the Tier 1 configuration defined in ADR-33 §Implementation Guidelines (Tier 1 — HSQLDB Oracle Compatibility), with JDBC URL base `jdbc:hsqldb:mem:testdb;sql.syntax_ora=true`.
- **BR-001-03**: All sample table schemas **MUST** use Oracle-compatible SQL syntax (e.g., `NUMBER`, `VARCHAR2`, `DATE`, `TIMESTAMP` types; `CLOB`/`BLOB` appear only in the type enforcement test table, BR-001-11).
- **BR-001-04**: Schema and seed data **MUST** be delivered as Flyway migrations located under `src/test/resources/db/migration/` and applied automatically per ADR-14 and ADR-33 §Decision (Tier 1).
- **BR-001-05**: Sample data scripts **MUST** populate tables with sufficient rows to validate look-ahead pagination (minimum 50 rows per primary table).
- **BR-001-06**: Sample data **MUST** cover all supported column types: `VARCHAR2` (string), `NUMBER` (integer and decimal), `FLOAT` (double), `NUMBER(1)` (boolean equivalent), `TIMESTAMP` (datetime), `DATE` (date).
- **BR-001-07**: Sample data **MUST** include NULL values for nullable columns to validate null handling serialization.
- **BR-001-08**: Sample data **MUST** include rows that exercise all filter types: date range filtering, boolean filtering, and dynamic key-value lookup filtering.
- **BR-001-09**: A dedicated lookup table **MUST** be provided for `dynamic_kv` filter validation with both active and inactive entries.
- **BR-001-10**: Schema definitions **MUST NOT** include complex/nested types (JSON, BLOB, CLOB, ARRAY) in primary query tables — those are tested separately via the type enforcement guard.
- **BR-001-11**: A separate type enforcement test table **MUST** be provided containing columns with disallowed types (CLOB, BLOB) to validate the security abort mechanism.
- **BR-001-12**: BigDecimal precision test data **MUST** include values with varying decimal scales (0, 2, 4, and 8 decimal places) to validate string serialization accuracy.
- **BR-001-13**: Date and timestamp test data **MUST** include boundary values: epoch start, year 2000, current year, and future dates.
- **BR-001-14**: The HSQLDB datasource **MUST** be injectable via Quarkus CDI using the same `datasource` name referenced in dashboard YAML configurations.
- **BR-001-15**: Tier 1 (HSQLDB) test infrastructure **MUST NOT** require Docker, network access, or any external service to execute. Oracle parity validation (Tier 2) is governed by ADR-33.
- **BR-001-16**: Seed data scripts **MUST** be idempotent and **MUST** use `MERGE` (Oracle-compatible). Schema migration immutability follows ADR-14; the schema DDL itself **MUST** remain Oracle-compatible per BR-001-03.
- **BR-001-17**: Test schema generation **MUST** follow ADR-33 (Flyway only, Hibernate DDL generation disabled).
- **BR-001-18**: Tests that use the HSQLDB datasource **MUST** be classified, tagged, named and executed as integration tests per ADR-32 §2 (Test Classification) and ADR-33 §Decision; domain unit tests **MUST NOT** use HSQLDB.

## 3. Acceptance Criteria / Scenarios

```gherkin
# BR-001-01, BR-001-02, BR-001-03
Scenario: Oracle compatibility mode is active
  Given the %test profile datasource is configured with sql.syntax_ora=true
  When a test suite connects to the in-memory database
  Then Oracle-compatible SQL syntax (NUMBER, VARCHAR2, DATE, TIMESTAMP) executes without errors

# BR-001-04, BR-001-17
Scenario: Flyway migrations initialize the schema before each test class
  Given Flyway migrations are placed in src/test/resources/db/migration/
  When a test class starts under the ADR-33 Tier 1 configuration
  Then all sample tables exist with the defined Oracle-compatible column types
  And no manual intervention was required

# BR-001-05
Scenario: Seed data supports look-ahead pagination boundaries
  Given a primary table contains 50 seeded rows
  When a pagination test requests offset=0 limit=20 with look-ahead limit+1
  Then 21 rows are returned and the pagination logic is exercised correctly

# BR-001-06, BR-001-07
Scenario: All column types and null handling are represented
  Given the seed data covers VARCHAR2, NUMBER, FLOAT, NUMBER(1), TIMESTAMP, and DATE columns
  When a row containing NULL values for nullable columns is serialized
  Then the null values are represented without serialization errors

# BR-001-10, BR-001-11
Scenario: Type enforcement guard rejects disallowed column types
  Given the type enforcement test table contains CLOB and BLOB columns
  When the security guard inspects ResultSetMetaData
  Then the disallowed types are detected and execution aborts

# BR-001-15, BR-001-18
Scenario: Tier 1 test infrastructure runs without external dependencies
  Given a workstation with no Docker daemon and no network access to databases
  When the Tier 1 (HSQLDB) integration tests execute
  Then all tests complete successfully using the JVM-embedded HSQLDB
  And every such test is classified as an integration test per ADR-32/ADR-33
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Configuration | ADR-15 | HSQLDB datasource configured in %test profile of application.properties | Yes |
| Architecture | ADR-10 | Test datasource injected via same port interface used in production | Yes |
| Data Migration | ADR-14 | Schema and seed data delivered as Flyway migrations (BR-001-04, BR-001-16) | TBD |
| Code Quality | ADR-12 | Test classes follow Google Java Style, validated by Spotless | Yes |
| Logging | ADR-16 | Test execution logs SQL statements at DEBUG level | Yes |
| Security | ADR-18 | Type enforcement guard validated against HSQLDB metadata | Yes |
| Build | ADR-03 | HSQLDB dependency scoped to `test` in Maven POM | Yes |
| Testing | ADR-32, ADR-33 | HSQLDB as Tier 1; Oracle parity is Tier 2 per ADR-33 (BR-001-18) | TBD |
| Comment Standards | ADR-34 | Traceability grammar per ADR-34 | TBD |

> **Evidence status (ADR-30 §Decision)**: ADR-30 requires satisfaction evidence in `requirements/BR-001-*/logs/`. That folder does not exist yet; checklist rows previously marked `Yes` are reset to `TBD` where a rule changed, and the remainder are unverified until test reports and logs are stored there.

## 5. Related ADRs

- ADR-03: Build Automation — HSQLDB test dependency declaration in parent POM
- ADR-07: Containerized Development Environment — Tier 2 Oracle parity via Dev Services; HSQLDB covers Tier 1 only
- ADR-12: Code Quality Standards — Spotless on test sources
- ADR-16: Structured Logging Strategy — SQL statement logging at DEBUG in tests
- ADR-18: Security Implementation (OWASP) — Type enforcement guard validated against HSQLDB metadata
- ADR-10: Software Architecture (Hexagonal) — Datasource adapter swapped for HSQLDB in test
- ADR-14: Data Migration Strategy — Test schema scripts follow Flyway naming patterns
- ADR-15: Application Configuration Management — %test profile datasource configuration
- ADR-32: Unit Testing Strategy — HSQLDB-backed tests are integration-tier (`@QuarkusTest`, `@Tag("integration")`); they are not context-free unit tests
- ADR-33: Integration Testing Strategy — HSQLDB is the Tier 1 database for adapter tests; Oracle is Tier 2
- ADR-34: Code Comment Standards — Traceability comment grammar
