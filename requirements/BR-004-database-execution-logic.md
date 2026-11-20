# BR-004: Database Execution Logic

## 1. Summary

As a backend engineer, I want a secure SQL execution pipeline with type enforcement, parameterized queries, and look-ahead pagination, so that dashboard queries execute safely without SQL injection risks, complex data leakage, or expensive COUNT(*) operations.

This feature defines the core backend execution pipelines including SQL injection mitigation via parameterized binding, a type enforcement guard that rejects non-scalar database types, look-ahead pagination (Zalando style) that avoids aggregate counts, and the phased implementation roadmap.

## 2. Business Rules

- **BR-004-01**: SQL queries **MUST** use parameterized binding via `query.setParameter(name, value)`; raw string concatenation of user inputs **SHALL NOT** be used.
- **BR-004-02**: The YAML SQL template **MUST** be treated as an invariant compilation block; runtime user inputs **MUST** be bound as parameters only.
- **BR-004-03**: The type enforcement guard **MUST** inspect `ResultSetMetaData.getColumnType()` before extracting any row data.
- **BR-004-04**: The following JDBC types **MUST** be rejected with an immediate execution abort and HTTP 500 response (error code `QUERY-SECURITY-001`, BR-003-24): `Types.OTHER` (JSON/JSONB), `Types.STRUCT`, `Types.BLOB`, `Types.CLOB`, `Types.ARRAY`.
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
- **BR-004-19**: BigDecimal values **MUST** be serialized as JSON strings via a custom Jackson serializer registered through a Quarkus `ObjectMapperCustomizer` CDI bean (e.g., Jackson `ToStringSerializer` for `BigDecimal`). A global `quarkus.jackson.*` property **MUST NOT** be relied upon for this rule: no such property exists, and Jackson's `WRITE_BIGDECIMAL_AS_PLAIN` still emits a JSON number, not a string (https://quarkus.io/guides/rest-json).
- **BR-004-20**: Database exception details **MUST** be written to infrastructure logs only; callers **MUST** receive the sanitized problem response defined by BR-003-13 and BR-003-17. Exception translation follows ADR-17.
- **BR-004-21**: Bound parameter values and result rows **MUST NOT** be logged or attached to spans/metrics; log content rules follow ADR-16.
- **BR-004-22**: Query execution **MUST** be a domain port implemented by an infrastructure adapter per ADR-10 and ADR-11; the domain layer **MUST NOT** depend on JDBC or `EntityManager` types.
- **BR-004-23**: Log and error messages **MUST** be externalized per ADR-26.

## 3. Acceptance Criteria / Scenarios

```gherkin
# BR-004-01, BR-004-02
Scenario: Parameterized query prevents SQL injection
  Given a dashboard SQL template with named parameter ":channel_filter"
  When a user submits filterInputs with channel_filter="'; DROP TABLE users; --"
  Then the value is bound as a parameter via setParameter
  And the SQL template is not modified
  And no SQL injection occurs

# BR-004-03, BR-004-04
Scenario: Type enforcement rejects JSON column
  Given a database table has a column of type JSONB
  When the query executes and ResultSetMetaData reports Types.OTHER
  Then execution aborts immediately
  And a 500 error response is returned with errorCode "QUERY-SECURITY-001" and a generic message

# BR-004-03, BR-004-04
Scenario: Type enforcement rejects BLOB column
  Given a database table has a column of type BLOB
  When the query executes and ResultSetMetaData reports Types.BLOB
  Then execution aborts immediately
  And a 500 error response is returned

# BR-004-07, BR-004-19
Scenario: BigDecimal serialized as plain string
  Given a column of type DECIMAL contains value 12450.75
  When the row is serialized to JSON
  Then the value appears as "12450.75" (string type, not 12450.75 number)

# BR-004-12, BR-004-13, BR-004-14, BR-004-17
Scenario: Look-ahead pagination detects more results
  Given the database has 50 matching rows
  When I request offset=0, limit=20
  Then the backend executes with setMaxResults(21)
  And 21 rows are returned from the database
  And the response contains 20 items
  And the response links include "next" with offset=20

# BR-004-15, BR-004-17
Scenario: Look-ahead pagination detects end of results
  Given the database has 35 matching rows
  When I request offset=20, limit=20
  Then the backend executes with setMaxResults(21)
  And 15 rows are returned from the database
  And the response contains 15 items
  And the response links do not include "next"

# BR-004-16
Scenario: Offset applied correctly
  Given the database has 100 matching rows
  When I request offset=40, limit=20
  Then setFirstResult(40) is applied
  And rows 41-60 are returned (0-indexed: 40-59)

# BR-004-20, BR-004-21
Scenario: Database error logged but not exposed
  Given a database connection timeout occurs
  When the query execution fails
  Then the exception and SQL template text are logged at ERROR level without bound parameter values
  And the API response status is 504 with a generic message, "errorCode" and "correlationId"
  And no SQL query text appears in the response body
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Security | ADR-18 | Parameterized queries prevent SQL injection; type guard blocks complex type leakage | TBD |
| Logging | ADR-16 | Structured logs without bound values or rows (BR-004-21) | TBD |
| i18n | ADR-25, ADR-26 | Externalized messages (BR-004-23) | TBD |
| Observability | ADR-20, ADR-21, ADR-22 | Query execution spans and metrics | TBD |
| Error Handling | ADR-17, ADR-23 | Exception translation and sanitized responses (BR-004-20) | TBD |
| Configuration | ADR-15 | Datasource/Agroal configuration only; BigDecimal-as-string via an `ObjectMapperCustomizer` CDI bean, not a property (BR-004-19) | TBD |
| Architecture | ADR-09, ADR-10, ADR-11 | Port/adapter placement (BR-004-22) | TBD |
| Code Quality | ADR-12 | Quality gates per ADR-12 | TBD |
| Testing | ADR-32, ADR-33 | Unit tier for mapping/pagination logic; integration tier on HSQLDB (BR-001) | TBD |
| Comment Standards | ADR-34 | Traceability grammar per ADR-34 | TBD |

## 5. Related ADRs

- ADR-08: Application Framework (Quarkus) — RESTEasy Reactive, Agroal datasource
- ADR-09: Multi-Module Structure — Module placement of domain vs. infrastructure code
- ADR-10: Software Architecture (Hexagonal) — Domain service and adapter pattern
- ADR-11: Package Structure Organization — `*Port` / `*Adapter` naming
- ADR-12: Code Quality Standards — Formatting and static analysis
- ADR-15: Application Configuration Management — Datasource configuration
- ADR-16: Structured Logging Strategy — Error detail logging and sanitization
- ADR-17: Exception Handling (Reactive) — Infrastructure exception translation and error codes
- ADR-18: Security Implementation (OWASP) — SQL injection prevention, type enforcement
- ADR-20: Observability Strategy — Query execution metrics
- ADR-21: Distributed Tracing — Execution span instrumentation
- ADR-22: Application Monitoring & Observability — Execution metrics
- ADR-23: REST API Error Handling — Error sanitization and RFC 9457 format
- ADR-25: Internationalization Strategy — Localized log/error messages
- ADR-26: Message Externalization — `LocalizedLogger`, `MessageKeys`
- ADR-32: Unit Testing Strategy — Unit tier for mapping and pagination logic
- ADR-33: Integration Testing Strategy — Execution tests on HSQLDB (Tier 1) and Oracle (Tier 2)
- ADR-34: Code Comment Standards — Traceability comments

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

- **Task 1.1:** Initialize Quarkus infrastructure with Java 21 LTS; add `quarkus-rest`, `quarkus-rest-jackson`, and `quarkus-agroal` dependencies (Quarkus 3.x artifact names per ADR-08).
- **Task 1.2:** Add YAML parsing dependency: `com.fasterxml.jackson.dataformat:jackson-dataformat-yaml`.
- **Task 1.3:** Build `DashboardSchema` POJO mapping YAML block literals to multi-line string properties.
- **Task 1.4:** Build `GitDashboardRegistryAdapter` (implements `DashboardRegistryPort`) with file observer threads loading configurations into `ConcurrentHashMap`.

### Phase 2: Secure Query Execution & Guardrails

- **Task 2.1:** Define `QueryExecutionPort` in the domain and build `JdbcQueryExecutionAdapter` in `platform-infrastructure` injecting `EntityManager` for database routing (BR-004-22).
- **Task 2.2:** Implement parameter binding mapping filter inputs to `query.setParameter(key, value)`.
- **Task 2.3:** Build `TypeEnforcementFilter` using `ResultSetMetaData.getColumnType()` to abort on non-primitive types.
- **Task 2.4:** Register an `ObjectMapperCustomizer` CDI bean applying `ToStringSerializer` to `BigDecimal` (BR-004-19; decimal-as-string contract shared with BR-003-09).

### Phase 3: Look-Ahead Database Pagination

- **Task 3.1:** Apply offset via `query.setFirstResult(offset)`.
- **Task 3.2:** Apply look-ahead limit via `query.setMaxResults(limit + 1)`.
- **Task 3.3:** Build the hypermedia link builder in the web adapter layer (`infrastructure.web.*`) to consume `hasNext` and generate navigation links.

### Phase 4: Data Masking Middleware

- **Task 4.1:** Build `DataMaskingMiddleware` post-processor to apply masking before serialization.
- **Task 4.2:** Implement `partial_text` and `suffix_only` masking strategies.
- **Task 4.3:** Implement `EmailMaskingStrategy` for email domain-preserving mask.

### Phase 5: Production Hardening & Global RFC 9457 Exceptions

- **Task 5.1:** Register global `ExceptionMapper<Throwable>` for all unhandled exceptions.
- **Task 5.2:** Build `ProblemResponseFilter` ensuring all errors output `application/problem+json`.
- **Task 5.3:** Build database exception translators mapping `SQLException` to ADR-17 infrastructure exceptions (`INFRA-INFRASTRUCTURE-002`/`-003`), routing detail to logs and returning `errorCode`/`correlationId` to callers.
