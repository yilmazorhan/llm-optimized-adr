# BR-004: Database Execution Logic

## 1. Summary

As a backend engineer, I want a secure SQL execution pipeline with type enforcement, parameterized queries, and look-ahead pagination, so that dashboard queries execute safely without SQL injection risks, complex data leakage, or expensive COUNT(*) operations.

This feature defines the core backend execution pipelines including SQL injection mitigation via parameterized binding, a type enforcement guard that rejects non-scalar database types, look-ahead pagination (Zalando style) that avoids aggregate counts, and the phased implementation roadmap.

## 2. Business Rules

- **BR-004-01**: SQL queries **MUST** use parameterized binding via `query.setParameter(name, value)`; raw string concatenation of user inputs **SHALL NOT** be used.
- **BR-004-02**: The YAML SQL template **MUST** be treated as an invariant compilation block; runtime user inputs **MUST** be bound as parameters only.
- **BR-004-03**: The type enforcement guard **MUST** inspect `ResultSetMetaData.getColumnType()` before extracting any row data.
- **BR-004-04**: The following JDBC types **MUST** be rejected with an immediate execution abort and HTTP 500 response: `Types.OTHER` (JSON/JSONB), `Types.STRUCT`, `Types.BLOB`, `Types.CLOB`, `Types.ARRAY`.
- **BR-004-05**: VARCHAR/TEXT columns **MUST** map to `java.lang.String` and serialize as JSON string.
- **BR-004-06**: BIGINT/INT columns **MUST** map to `java.lang.Long` or `java.lang.Integer` and serialize as JSON integer (int64).
- **BR-004-07**: DECIMAL/NUMERIC columns **MUST** map to `java.math.BigDecimal` and serialize as JSON string to prevent IEEE-754 precision loss.
- **BR-004-08**: DOUBLE/FLOAT columns **MUST** map to `java.lang.Double` and serialize as JSON number (double).
- **BR-004-09**: BOOLEAN/BIT columns **MUST** map to `java.lang.Boolean` and serialize as JSON boolean.
- **BR-004-10**: TIMESTAMP/UTC columns **MUST** map to `java.time.Instant` and serialize as ISO-8601 date-time string.
- **BR-004-11**: DATE columns **MUST** map to `java.time.LocalDate` and serialize as ISO-8601 date string.
- **BR-004-12**: Pagination **MUST** use look-ahead strategy; the backend **SHALL NOT** execute `COUNT(*)` aggregate queries.
- **BR-004-13**: The backend **MUST** request `limit + 1` rows from the database (`setMaxResults(limit + 1)`).
- **BR-004-14**: If `limit + 1` rows are returned, the pipeline **MUST** set `hasNext = true`, remove the extra row, and return only `limit` rows.
- **BR-004-15**: If `limit` or fewer rows are returned, the pipeline **MUST** set `hasNext = false` and return all available rows.
- **BR-004-16**: Pagination offset **MUST** be applied via `query.setFirstResult(offset)`.
- **BR-004-17**: The hypermedia response builder **MUST** generate `next` link only when `hasNext = true`.
- **BR-004-18**: The hypermedia response builder **MUST** generate `prev` link only when `offset > 0`.
- **BR-004-19**: Jackson ObjectMapper **MUST** be configured with `quarkus.jackson.write-bigdecimals-as-plain-strings=true` globally.
- **BR-004-20**: Database exception details **MUST** be written to infrastructure logs only; sanitized error tokens **MUST** be returned to API callers.

## 3. Acceptance Criteria / Scenarios

```gherkin
Scenario: Parameterized query prevents SQL injection
  Given a dashboard SQL template with named parameter ":channel_filter"
  When a user submits filter_inputs with channel_filter="'; DROP TABLE users; --"
  Then the value is bound as a parameter via setParameter
  And the SQL template is not modified
  And no SQL injection occurs

Scenario: Type enforcement rejects JSON column
  Given a database table has a column of type JSONB
  When the query executes and ResultSetMetaData reports Types.OTHER
  Then execution aborts immediately
  And a 500 error response is returned with a security violation message

Scenario: Type enforcement rejects BLOB column
  Given a database table has a column of type BLOB
  When the query executes and ResultSetMetaData reports Types.BLOB
  Then execution aborts immediately
  And a 500 error response is returned

Scenario: BigDecimal serialized as plain string
  Given a column of type DECIMAL contains value 12450.75
  When the row is serialized to JSON
  Then the value appears as "12450.75" (string type, not 12450.75 number)

Scenario: Look-ahead pagination detects more results
  Given the database has 50 matching rows
  When I request offset=0, limit=20
  Then the backend executes with setMaxResults(21)
  And 21 rows are returned from the database
  And the response contains 20 items
  And the response links include "next" with offset=20

Scenario: Look-ahead pagination detects end of results
  Given the database has 35 matching rows
  When I request offset=20, limit=20
  Then the backend executes with setMaxResults(21)
  And 15 rows are returned from the database
  And the response contains 15 items
  And the response links do not include "next"

Scenario: Offset applied correctly
  Given the database has 100 matching rows
  When I request offset=40, limit=20
  Then setFirstResult(40) is applied
  And rows 41-60 are returned (0-indexed: 40-59)

Scenario: Database error logged but not exposed
  Given a database connection timeout occurs
  When the query execution fails
  Then the full exception with SQL text is logged at ERROR level
  And the API response contains only a generic "internal error" message
  And no SQL query text appears in the response body
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Security | ADR-18 | Parameterized queries prevent SQL injection; type guard blocks complex type leakage | TBD |
| Logging | ADR-16 | Query execution metrics and errors logged with structured JSON; SQL text in logs only | TBD |
| Observability | ADR-20, ADR-21 | Spans for query execution, parameter binding, type checking, pagination | TBD |
| Error Handling | ADR-23 | Database errors sanitized to RFC 9457 responses | TBD |
| Configuration | ADR-15 | Jackson BigDecimal serialization configured in application.properties | TBD |
| Architecture | ADR-10 | Query execution service in domain layer; database access via port/adapter | TBD |

## 5. Related ADRs

- ADR-08: Application Framework (Quarkus) — RESTEasy Reactive, Agroal datasource
- ADR-10: Software Architecture (Hexagonal) — Domain service and adapter pattern
- ADR-15: Application Configuration Management — Jackson configuration properties
- ADR-16: Structured Logging Strategy — Error detail logging
- ADR-18: Security Implementation (OWASP) — SQL injection prevention, type enforcement
- ADR-20: Observability Strategy — Query execution metrics
- ADR-21: Distributed Tracing — Execution span instrumentation
- ADR-23: REST API Error Handling — Error sanitization and RFC 9457 format

## 6. Scope

### In-Scope

- SQL injection mitigation via parameterized binding
- Database-to-Java type mapping matrix
- Type enforcement guard (reject JSON, STRUCT, BLOB, CLOB, ARRAY)
- Look-ahead pagination (Zalando style, no COUNT(*))
- Hypermedia link generation (self, prev, next)
- BigDecimal string serialization configuration
- Error sanitization (no internal details in responses)
- Implementation roadmap (5 phases)

### Out-of-Scope

- Dashboard YAML format definition (see BR-002)
- REST API endpoint schema (see BR-003)
- Data masking strategy implementation (see BR-002)
- Database schema design
- Connection pooling configuration

## 7. Type Mapping Matrix

| Database Type | Java (Quarkus) Class | JSON Serialization | OpenAPI Format | Boundary Strategy |
|---|---|---|---|---|
| VARCHAR / TEXT | java.lang.String | string | None | Plain output conversion |
| BIGINT / INT | java.lang.Long / Integer | integer | int64 | Native numeric encoding |
| DECIMAL / NUMERIC | java.math.BigDecimal | string | decimal | Forced as plain text via Jackson |
| DOUBLE / FLOAT | java.lang.Double | number | double | Native floating-point notation |
| BOOLEAN / BIT | java.lang.Boolean | boolean | None | Native boolean flags |
| TIMESTAMP / UTC | java.time.Instant | string | date-time | ISO-8601 encoding |
| DATE | java.time.LocalDate | string | date | ISO-8601 date string |
| JSON / JSONB | *Rejected* | *Blocked* | *Excluded* | **Forced Abort (Security Exception)** |

## 8. Implementation Phases

### Phase 1: Core Framework & YAML Processing Pipeline

- **Task 1.1:** Initialize Quarkus infrastructure with Java 21 LTS; add `quarkus-resteasy-reactive`, `quarkus-resteasy-reactive-jackson`, and `quarkus-agroal` dependencies.
- **Task 1.2:** Add YAML parsing dependency: `com.fasterxml.jackson.dataformat:jackson-dataformat-yaml`.
- **Task 1.3:** Build `DashboardSchema` POJO mapping YAML block literals to multi-line string properties.
- **Task 1.4:** Build `GitSchemaRegistry` with file observer threads loading configurations into `ConcurrentHashMap`.

### Phase 2: Secure Query Execution & Guardrails

- **Task 2.1:** Build `QueryExecutionService` injecting `EntityManager` for database routing.
- **Task 2.2:** Implement parameter binding mapping filter inputs to `query.setParameter(key, value)`.
- **Task 2.3:** Build `TypeEnforcementFilter` using `ResultSetMetaData.getColumnType()` to abort on non-primitive types.
- **Task 2.4:** Configure Jackson: `quarkus.jackson.write-bigdecimals-as-plain-strings=true`.

### Phase 3: Look-Ahead Database Pagination

- **Task 3.1:** Apply offset via `query.setFirstResult(offset)`.
- **Task 3.2:** Apply look-ahead limit via `query.setMaxResults(limit + 1)`.
- **Task 3.3:** Build `HypermediaResponseFilter` to detect extra row, set `hasNext`, and generate navigation links.

### Phase 4: Data Masking Middleware

- **Task 4.1:** Build `DataMaskingMiddleware` post-processor to apply masking before serialization.
- **Task 4.2:** Implement `partial_text` and `suffix_only` masking strategies.
- **Task 4.3:** Implement `EmailMaskingStrategy` for email domain-preserving mask.

### Phase 5: Production Hardening & Global RFC 9457 Exceptions

- **Task 5.1:** Register global `ExceptionMapper<Throwable>` for all unhandled exceptions.
- **Task 5.2:** Build `ProblemResponseFilter` ensuring all errors output `application/problem+json`.
- **Task 5.3:** Build database exception sanitizers routing detail to logs, returning safe tokens to callers.
