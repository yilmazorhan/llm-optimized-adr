# BR-003: REST API Query Endpoint

## 1. Summary

As a reporting consumer, I want a standardized REST API endpoint to execute dashboard queries with filters, pagination, and masked data, so that I can retrieve structured, secure reporting data without direct database access.

This feature defines a single POST endpoint (`/api/dashboards/{dashboardId}/query`) that loads the YAML dashboard configuration from cache, binds filter parameters securely, executes look-ahead pagination, applies data masking, and returns sanitized rows with hypermedia navigation links. All errors follow the RFC 9457 Problem Details specification. JSON property names follow camelCase (ADR-24); the YAML `dashboard_id` key defined in BR-002 is a configuration-file key and is exposed in the API as `dashboardId`.

## 2. Business Rules

- **BR-003-01**: The API **MUST** expose a single endpoint: `POST /api/dashboards/{dashboardId}/query`.
- **BR-003-02**: The `dashboardId` path parameter **MUST** map to a valid Git-backed YAML dashboard configuration.
- **BR-003-03**: The request body **MUST** contain `offset` (integer, minimum 0) and `limit` (integer, minimum 1, maximum 100) properties.
- **BR-003-04**: The request body **MAY** contain a `filterInputs` object with key-value pairs matching the dashboard's defined filter names.
- **BR-003-05**: Filter input values **MUST** be flat primitive types only; nested objects or arrays **SHALL NOT** be accepted.
- **BR-003-06**: The response **MUST** include `dashboardId`, `links` (hypermedia navigation), and `items` (array of flat row objects).
- **BR-003-07**: Hypermedia links **MUST** include a `self` link; `prev` and `next` links **SHALL** be included when applicable based on pagination state. Because the endpoint is POST (a link URL cannot carry the request body), `prev`/`next` **MUST** be represented as pagination continuation descriptors — each **MUST** contain `offset`, `limit`, and the endpoint path — which the client submits as the next request body's `offset`/`limit` (filter inputs are re-supplied by the client unchanged).
- **BR-003-08**: Each row in `items` **MUST** contain only flat primitive values corresponding to the dashboard column definitions.
- **BR-003-09**: Decimal/monetary values **MUST** be serialized as strings (not floating-point numbers) to prevent IEEE-754 precision loss in browser runtimes.
- **BR-003-10**: Date-time values **MUST** be serialized in ISO-8601 UTC format (`yyyy-MM-dd'T'HH:mm:ss'Z'`).
- **BR-003-11**: Date values **MUST** be serialized in ISO-8601 format (`yyyy-MM-dd`).
- **BR-003-12**: All error responses **MUST** use `application/problem+json` content type following RFC 9457.
- **BR-003-13**: Error response bodies **MUST** conform to the problem detail model, field requirements, `type` URI convention and timestamp format defined in ADR-23 §Decision, and **MUST** carry a registered error code and the correlation identifier (ADR-17, ADR-16).
- **BR-003-14**: Internal contextual details (query text, connection parameters, index keys) **SHALL NOT** be exposed in error responses; they **MUST** be routed to secure system logs only.
- **BR-003-15**: A request referencing a non-existent `dashboardId` **MUST** return HTTP 404.
- **BR-003-16**: A request with invalid filter parameter types, missing required filters, or `offset`/`limit` violations of BR-003-03 **MUST** return HTTP 400 with field-level validation errors as specified in ADR-23 and ADR-24.
- **BR-003-17**: Database unavailability and database timeout **MUST** return the retryable statuses defined by the ADR-23 error-code-to-status mapping (503/504); unsupported column type detection and any other internal execution fault **MUST** return HTTP 500.
- **BR-003-18**: Nullable column values **MUST** be represented as explicit JSON `null`.
- **BR-003-19**: The endpoint **MUST** be secured as specified in ADR-18 §Decision (authentication, RBAC at the inbound adapter); authentication and authorization failures **MUST** return HTTP 401/403 as `application/problem+json` (ADR-23), and security **MUST NOT** be disabled in integration tests (ADR-33 §Decision).
- **BR-003-20**: The endpoint **MUST** be rate limited as specified in ADR-18 §Decision; an exceeded limit **MUST** return HTTP 429 as `application/problem+json`.
- **BR-003-21**: Query and error responses **MUST** carry the security headers defined in ADR-18 and **MUST NOT** be cacheable, because they contain sensitive data (masked, but derived from PII sources).
- **BR-003-22**: Correlation identifier handling **MUST** follow ADR-16 §Decision (request/response header, MDC, error body).
- **BR-003-23**: Error `title` and `detail` **MUST** be localized through message bundles selected by `Accept-Language` as specified in ADR-25 and ADR-26; `errorCode` values **MUST NOT** be localized.
- **BR-003-24**: Each error condition **MUST** map to a registered error code (proposed, to be registered in the ADR-17 registry and the ADR-31 error inventory): 404 `DASHBOARD-BUSINESS-001`; 400 `QUERY-VALIDATION-001`; 401 `QUERY-AUTHORIZATION-002`; 403 `QUERY-AUTHORIZATION-001`; 429 `QUERY-BUSINESS-001`; 503 `INFRA-INFRASTRUCTURE-002`; 504 `INFRA-INFRASTRUCTURE-003`; 500 unsupported column type `QUERY-SECURITY-001`; 500 other `QUERY-INFRASTRUCTURE-001`.
- **BR-003-25**: Each query execution **MUST** be audited per ADR-19 §Decision; filter values **MUST** be handled under the ADR-19 PII masking rules.
- **BR-003-26**: Bound parameter values, `filterInputs` values and response rows **MUST NOT** appear in logs, span attributes or metric labels (ADR-16, ADR-20, ADR-21 content rules).
- **BR-003-27**: Query count, latency and error metrics **MUST** be exposed and trace sampling configured as specified in ADR-22 and ADR-21.
- **BR-003-28**: The OpenAPI specification for this endpoint **MUST** be produced and localized as specified in ADR-27 §Decision and published as the ADR-31 OpenAPI artifact, declaring the JWT security scheme required by BR-003-19.

## 3. Acceptance Criteria / Scenarios

```gherkin
# BR-003-01, BR-003-06, BR-003-19, BR-003-21
Scenario: Successful query execution with filters
  Given the dashboard "merchant_settlement_ledger" is configured and cached
  And I hold a valid JWT with an authorized role
  When I POST to "/api/dashboards/merchant_settlement_ledger/query" with offset=0, limit=20, and filterInputs containing channel_filter="BANK_TRANSFER"
  Then the response status is 200
  And the response content-type is "application/json"
  And the response carries "X-Correlation-Id" and "Cache-Control: no-cache, no-store, max-age=0" headers
  And the response body contains "dashboardId" = "merchant_settlement_ledger"
  And the response body contains "items" array with up to 20 rows
  And the response body contains "links.self" with the current pagination state

# BR-003-07
Scenario: Pagination with next link
  Given the dashboard query returns more than 20 results
  When I POST with offset=0, limit=20
  Then the response contains "links.next" pointing to offset=20
  And the "items" array contains exactly 20 rows

# BR-003-07
Scenario: Pagination at end of results
  Given the dashboard query returns exactly 15 results for offset=40
  When I POST with offset=40, limit=20
  Then the response does not contain "links.next"
  And the "items" array contains 15 rows

# BR-003-07
Scenario: Previous link included for non-zero offset
  Given offset=20 is requested
  When the query executes successfully
  Then the response contains "links.prev" pointing to offset=0

# BR-003-12, BR-003-13, BR-003-15, BR-003-24
Scenario: Non-existent dashboard returns 404
  Given no dashboard configuration exists for "nonexistent_dashboard"
  When I POST to "/api/dashboards/nonexistent_dashboard/query"
  Then the response status is 404
  And the response content-type is "application/problem+json"
  And the response body contains "type", "title", "status", "detail", "instance", "timestamp", "errorCode", "correlationId"
  And "errorCode" is "DASHBOARD-BUSINESS-001"

# BR-003-16, BR-003-24
Scenario: Invalid filter type returns 400
  Given the dashboard expects filter "start_date" of type date
  When I POST with filterInputs containing start_date="2026/05/18" (invalid format)
  Then the response status is 400
  And the response content-type is "application/problem+json"
  And "errorCode" is "QUERY-VALIDATION-001"
  And the detail field describes the expected ISO-8601 format

# BR-003-16
Scenario: Missing required filter returns 400
  Given the dashboard defines "channel_filter" as required
  When I POST without providing "channel_filter" in filterInputs
  Then the response status is 400
  And the response describes the missing required filter

# BR-003-09
Scenario: Decimal values serialized as strings
  Given the dashboard column "net_amount" is type "decimal"
  When query results contain a BigDecimal value of 12450.75
  Then the JSON response serializes it as the string "12450.75"

# BR-003-14, BR-003-17
Scenario: Database unavailability returns 503 without leaking implementation details
  Given a database connection failure occurs
  When the query execution fails
  Then the response status is 503
  And the response content-type is "application/problem+json"
  And the response carries a "Retry-After" header
  And the detail field contains a generic error message
  And no SQL query text, connection strings, or stack traces are present in the response

# BR-003-17
Scenario: Unsupported column type returns 500
  Given the dashboard SQL returns a column of type CLOB
  When the query executes
  Then the response status is 500
  And "errorCode" is "QUERY-SECURITY-001"

# BR-003-19
Scenario: Missing token returns 401
  When I POST to the query endpoint without a Bearer token
  Then the response status is 401
  And the response content-type is "application/problem+json"

# BR-003-19
Scenario: Insufficient role returns 403
  Given I hold a valid JWT without an authorized role
  When I POST to the query endpoint
  Then the response status is 403
  And the response content-type is "application/problem+json"

# BR-003-20
Scenario: Rate limit exceeded returns 429
  Given I have exceeded the configured request rate
  When I POST to the query endpoint
  Then the response status is 429
  And the response carries a "Retry-After" header

# BR-003-23
Scenario: Error messages are localized
  Given the request header "Accept-Language: de"
  When I POST to "/api/dashboards/nonexistent_dashboard/query"
  Then "title" and "detail" are the German bundle texts for "DASHBOARD-BUSINESS-001"
  And "errorCode" remains "DASHBOARD-BUSINESS-001"

# BR-003-25, BR-003-26
Scenario: Query is audited without sensitive values
  When a query executes successfully
  Then an audit event with actor, dashboardId, outcome and correlationId is emitted
  And no filter values or response rows appear in logs, spans or metrics
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Security | ADR-18 | Authentication, RBAC, rate limiting, headers per ADR-18 (BR-003-19..21); input validation; no internal details in errors | TBD |
| Logging | ADR-16 | Correlation ID and log content rules (BR-003-22, BR-003-26) | TBD |
| i18n | ADR-25, ADR-26 | Localized error `title`/`detail` (BR-003-23) | TBD |
| Observability | ADR-20, ADR-21, ADR-22 | Query spans, metrics and sampling (BR-003-27) | TBD |
| Audit | ADR-19 | Query audit event (BR-003-25) | TBD |
| Error Handling | ADR-17, ADR-23 | Problem Details with registered error codes (BR-003-13, BR-003-24) | TBD |
| Validation | ADR-24 | Request DTO validated with Bean Validation annotations | TBD |
| API Documentation | ADR-27, ADR-31 | Localized OpenAPI documentation and YAML artifact (BR-003-28) | TBD |
| Architecture | ADR-09, ADR-10, ADR-11 | Controller as inbound adapter; placement per ADR-11 | TBD |
| Configuration | ADR-15 | Error handling, rate limit and header properties per ADR-15 | TBD |
| Code Quality | ADR-12 | Quality gates per ADR-12 | TBD |
| Testing | ADR-32, ADR-33 | Endpoint integration tests via REST Assured `@QuarkusTest`; `problem+json` (with `errorCode`) and hypermedia contract assertions | TBD |
| Comment Standards | ADR-34 | Traceability comments per ADR-34 grammar | TBD |

## 5. Related ADRs

- ADR-08: Application Framework (Quarkus) — RESTEasy Reactive for endpoint implementation
- ADR-09: Multi-Module Structure — Web API module depends on business interfaces only
- ADR-10: Software Architecture (Hexagonal) — Controller as inbound adapter
- ADR-11: Package Structure Organization — `infrastructure.web.*` placement
- ADR-12: Code Quality Standards — Formatting and static analysis
- ADR-15: Application Configuration Management — Error handling, rate limit and header properties
- ADR-16: Structured Logging Strategy — Correlation ID and log content rules
- ADR-17: Exception Handling (Reactive) — Error code registry
- ADR-18: Security Implementation (OWASP) — JWT/RBAC, rate limiting, security headers, input validation, error sanitization
- ADR-19: Audit Logging Strategy — Query audit events
- ADR-20: Observability Strategy — Request tracing and metrics
- ADR-21: Distributed Tracing — OpenTelemetry span creation per query
- ADR-22: Application Monitoring & Observability — Micrometer metrics
- ADR-23: REST API Error Handling — RFC 9457 Problem Details format
- ADR-24: Data Transfer & Validation — DTO patterns with Bean Validation
- ADR-25: Internationalization Strategy — Localized error messages
- ADR-26: Message Externalization — Message bundles and `MessageKeys`
- ADR-27: API Documentation Localization — OpenAPI spec generation
- ADR-31: Production Build Artifacts — OpenAPI YAML and error inventory artifacts
- ADR-32: Unit Testing Strategy — Mapper and validator unit tests
- ADR-33: Integration Testing Strategy — REST Assured endpoint tests (hypermedia, masking, problem+json contracts)
- ADR-34: Code Comment Standards — `BR-003-NN` traceability comments in controller and mapper code

## 6. Scope

### In-Scope

- POST endpoint definition and request/response schemas
- Pagination request parameters (offset, limit)
- Filter input parameter binding
- Hypermedia navigation links (self, prev, next)
- Response data type serialization rules (decimal as string, ISO-8601 dates)
- JWT authentication, RBAC, rate limiting, and security headers for the endpoint
- RFC 9457 error response format for 400, 401, 403, 404, 429, 500, 503, and 504 errors
- Audit event emission per query
- Error detail sanitization (no internal leakage)

### Out-of-Scope

- Dashboard YAML configuration format (see BR-002)
- SQL execution engine and type enforcement (see BR-004)
- Look-ahead pagination implementation (see BR-004)
- Identity provider and token issuance (tokens are issued externally; key management per ADR-18)
- Frontend dashboard rendering

## 7. OpenAPI Specification

```json
{
  "openapi": "3.0.3",
  "info": {
    "title": "Dynamic Schema-Driven Reporting Backend Engine",
    "version": "1.0.0",
    "description": "Stateless, Git-configured backend query service featuring parameterized SQL execution, look-ahead pagination, strict primitive data masking, and RFC 9457 error handling."
  },
  "paths": {
    "/api/dashboards/{dashboardId}/query": {
      "post": {
        "summary": "Execute dynamic reporting dashboard query",
        "security": [{ "bearerAuth": [] }],
        "description": "Loads the YAML configuration from cache, binds parameters securely, executes look-ahead pagination, masks data, and returns sanitized rows.",
        "parameters": [
          {
            "name": "dashboardId",
            "in": "path",
            "required": true,
            "schema": { "type": "string" }
          }
        ],
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": { "$ref": "#/components/schemas/QueryRequest" }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Successful query response with hypermedia links and masked rows.",
            "content": {
              "application/json": {
                "schema": { "$ref": "#/components/schemas/QueryResponse" }
              }
            }
          },
          "400": {
            "description": "Invalid input, missing required parameters, or type mismatch.",
            "content": {
              "application/problem+json": {
                "schema": { "$ref": "#/components/schemas/ProblemDetails" }
              }
            }
          },
          "404": {
            "description": "Dashboard identifier not found in Git storage.",
            "content": {
              "application/problem+json": {
                "schema": { "$ref": "#/components/schemas/ProblemDetails" }
              }
            }
          },
          "401": {
            "description": "Missing or invalid JWT Bearer token.",
            "content": { "application/problem+json": { "schema": { "$ref": "#/components/schemas/ProblemDetails" } } }
          },
          "403": {
            "description": "Authenticated caller lacks the required role.",
            "content": { "application/problem+json": { "schema": { "$ref": "#/components/schemas/ProblemDetails" } } }
          },
          "429": {
            "description": "Rate limit exceeded (Retry-After header present).",
            "content": { "application/problem+json": { "schema": { "$ref": "#/components/schemas/ProblemDetails" } } }
          },
          "500": {
            "description": "Internal execution fault or unsupported column type detected.",
            "content": {
              "application/problem+json": {
                "schema": { "$ref": "#/components/schemas/ProblemDetails" }
              }
            }
          },
          "503": {
            "description": "Database unavailable (retryable, Retry-After header present).",
            "content": { "application/problem+json": { "schema": { "$ref": "#/components/schemas/ProblemDetails" } } }
          },
          "504": {
            "description": "Database timeout (retryable, Retry-After header present).",
            "content": { "application/problem+json": { "schema": { "$ref": "#/components/schemas/ProblemDetails" } } }
          }
        }
      }
    }
  },
  "components": {
    "securitySchemes": {
      "bearerAuth": { "type": "http", "scheme": "bearer", "bearerFormat": "JWT" }
    },
    "schemas": {
      "QueryRequest": {
        "type": "object",
        "required": ["offset", "limit"],
        "properties": {
          "offset": { "type": "integer", "minimum": 0, "example": 0 },
          "limit": { "type": "integer", "minimum": 1, "maximum": 100, "example": 20 },
          "filterInputs": { "type": "object", "additionalProperties": true }
        }
      },
      "QueryResponse": {
        "type": "object",
        "required": ["dashboardId", "links", "items"],
        "properties": {
          "dashboardId": { "type": "string" },
          "links": { "$ref": "#/components/schemas/HypermediaLinks" },
          "items": { "type": "array", "items": { "$ref": "#/components/schemas/Row" } }
        }
      },
      "HypermediaLinks": {
        "type": "object",
        "required": ["self"],
        "properties": {
          "self": { "type": "string" },
          "prev": { "$ref": "#/components/schemas/PaginationLink" },
          "next": { "$ref": "#/components/schemas/PaginationLink" }
        }
      },
      "PaginationLink": {
        "type": "object",
        "description": "Pagination continuation descriptor for a POST query endpoint (BR-003-07): client re-submits offset/limit with the same filter inputs.",
        "required": ["offset", "limit"],
        "properties": {
          "offset": { "type": "integer", "minimum": 0 },
          "limit": { "type": "integer", "minimum": 1, "maximum": 100 }
        }
      },
      "Row": {
        "type": "object",
        "additionalProperties": { "$ref": "#/components/schemas/ColumnValue" }
      },
      "ColumnValue": {
        "nullable": true,
        "oneOf": [
          { "type": "string" },
          { "type": "integer", "format": "int64" },
          { "type": "string", "format": "decimal" },
          { "type": "number", "format": "double" },
          { "type": "boolean" },
          { "type": "string", "format": "date-time" },
          { "type": "string", "format": "date" }
        ]
      },
      "ProblemDetails": {
        "type": "object",
        "required": ["type", "title", "status", "detail", "instance", "timestamp", "errorCode", "correlationId"],
        "properties": {
          "type": { "type": "string" },
          "title": { "type": "string" },
          "status": { "type": "integer" },
          "detail": { "type": "string" },
          "instance": { "type": "string" },
          "timestamp": { "type": "string", "format": "date-time", "description": "Pattern yyyy-MM-dd'T'HH:mm:ss.SSSXXX (ADR-23)" },
          "errorCode": { "type": "string", "description": "ADR-25 [DOMAIN]-[CATEGORY]-[SEQUENCE] error code (ADR-17 registry)" },
          "correlationId": { "type": "string", "description": "Correlation identifier (ADR-16); also returned as X-Correlation-Id header" },
          "applicationVersion": { "type": "string" },
          "retryable": { "type": "boolean" },
          "remediationSteps": { "type": "array", "items": { "type": "string" } },
          "traceId": { "type": "string", "description": "OpenTelemetry trace identifier (ADR-21)" },
          "spanId": { "type": "string", "description": "OpenTelemetry span identifier (ADR-21)" }
        }
      }
    }
  }
}
```

## 8. Error Response Examples

**HTTP 400 — Invalid Filter Parameter:**
```json
{
  "type": "https://api.reportingengine.internal/problems/QUERY-VALIDATION-001",
  "title": "Invalid Filter Parameter",
  "status": 400,
  "detail": "The filter 'start_date' expects a valid ISO-8601 calendar date format. Provided value: '2026/05/18'",
  "instance": "/api/dashboards/merchant_settlement_ledger/query",
  "timestamp": "2026-05-18T12:26:04.000Z",
  "errorCode": "QUERY-VALIDATION-001",
  "correlationId": "7f3c2a9e-5b1d-4c68-9a0e-2d4f6b8c1e33"
}
```

**HTTP 500 — Internal Execution Fault:**
```json
{
  "type": "https://api.reportingengine.internal/problems/QUERY-SECURITY-001",
  "title": "Unsupported Column Type Error",
  "status": 500,
  "detail": "An internal query mapping fault occurred. Unstructured or complex nested column schemas are blocked for security.",
  "instance": "/api/dashboards/merchant_settlement_ledger/query",
  "timestamp": "2026-05-18T12:26:04.000Z",
  "errorCode": "QUERY-SECURITY-001",
  "correlationId": "1b9d6bcd-bbfd-4b2d-9b5d-ab8dfbbd4bed"
}
```

> Error codes are proposed and must be registered per ADR-17 and listed in the ADR-31 error inventory.
