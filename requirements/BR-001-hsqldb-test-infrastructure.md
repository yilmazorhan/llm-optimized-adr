# BR-001: HSQLDB Test Infrastructure

## 1. Summary

As a **backend developer**, I want **an in-memory HSQLDB database configured with Oracle compatibility mode, pre-loaded sample table schemas, and realistic test data**, so that **I can validate every query function, type mapping, pagination logic, and masking pipeline locally without requiring a real database instance or Docker containers**.

This requirement establishes the test database layer that enables rapid, isolated validation of all service functions defined in BR-002, BR-003, and BR-004. HSQLDB running in Oracle syntax compatibility mode provides a zero-dependency, JVM-embedded database that supports the same SQL dialect patterns used in production Oracle environments. Sample schemas mirror the production dashboard query targets, and seed data covers all column types, filter combinations, edge cases, and pagination boundaries.

## 2. Business Rules

- **BR-001-01**: The test profile **MUST** use HSQLDB 2.7.x as the in-memory database engine with Oracle compatibility mode enabled (`sql.syntax_ora=true`).
- **BR-001-02**: HSQLDB **MUST** be configured as a Quarkus datasource in the `%test` profile with JDBC URL: `jdbc:hsqldb:mem:testdb;sql.syntax_ora=true`.
- **BR-001-03**: All sample table schemas **MUST** use Oracle-compatible SQL syntax (e.g., `NUMBER`, `VARCHAR2`, `DATE`, `TIMESTAMP`, `CLOB` types).
- **BR-001-04**: Schema initialization scripts **MUST** be placed in `src/test/resources/db/` and executed automatically before each test suite.
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
- **BR-001-15**: Test infrastructure **MUST NOT** require Docker, network access, or any external service to execute.
- **BR-001-16**: Schema and data scripts **MUST** be idempotent — safe to execute multiple times without errors (`CREATE TABLE IF NOT EXISTS` or equivalent, `MERGE` for data).

## 3. Acceptance Criteria / Scenarios

- HSQLDB **MUST** run in Oracle compatibility mode
- all tests **MUST** use in-memory HSQLDB

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Configuration | ADR-15 | HSQLDB datasource configured in %test profile of application.properties | Yes |
| Architecture | ADR-10 | Test datasource injected via same port interface used in production | Yes |
| Data Migration | ADR-14 | Test scripts use Flyway-compatible naming for schema init | Yes |
| Code Quality | ADR-12 | Test classes follow Google Java Style, validated by Spotless | Yes |
| Logging | ADR-16 | Test execution logs SQL statements at DEBUG level | Yes |
| Security | ADR-18 | Type enforcement guard validated against HSQLDB metadata | Yes |
| Build | ADR-03 | HSQLDB dependency scoped to `test` in Maven POM | Yes |

## 5. Related ADRs

- ADR-03: Build Automation — HSQLDB test dependency declaration in parent POM
- ADR-07: Containerized Development Environment — HSQLDB as Docker-free alternative for unit tests
- ADR-10: Software Architecture (Hexagonal) — Datasource adapter swapped for HSQLDB in test
- ADR-14: Data Migration Strategy — Test schema scripts follow Flyway naming patterns
- ADR-15: Application Configuration Management — %test profile datasource configuration
- ADR-23: REST API Error Handling — Error responses validated against in-memory data
- ADR-24: Data Transfer & Validation — DTO serialization validated with known test data
