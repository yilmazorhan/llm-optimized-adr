# BR-003: REST API Query Endpoint

## 1. Summary

As a reporting consumer, I want a standardized REST API endpoint to execute dashboard queries with filters, pagination, and masked data, so that I can retrieve structured, secure reporting data without direct database access.

This feature defines a single POST endpoint (`/api/dashboards/{dashboardId}/query`) that loads the YAML dashboard configuration from cache, binds filter parameters securely, executes look-ahead pagination, applies data masking, and returns sanitized rows with hypermedia navigation links. All errors follow the RFC 9457 Problem Details specification.

## 2. Business Rules

- **BR-003-01**: The API **MUST** expose a single endpoint: `POST /api/dashboards/{dashboardId}/query`.
- **BR-003-02**: The `dashboardId` path parameter **MUST** map to a valid Git-backed YAML dashboard configuration.
- **BR-003-03**: The request body **MUST** contain `offset` (integer, minimum 0) and `limit` (integer, minimum 1, maximum 100) properties.
- **BR-003-04**: The request body **MAY** contain a `filter_inputs` object with key-value pairs matching the dashboard's defined filter names.
- **BR-003-05**: Filter input values **MUST** be flat primitive types only; nested objects or arrays **SHALL NOT** be accepted.
- **BR-003-06**: The response **MUST** include `dashboard_id`, `links` (hypermedia navigation), and `items` (array of flat row objects).
- **BR-003-07**: Hypermedia links **MUST** include a `self` link; `prev` and `next` links **SHALL** be included when applicable based on pagination state.
- **BR-003-08**: Each row in `items` **MUST** contain only flat primitive values corresponding to the dashboard column definitions.
- **BR-003-09**: Decimal/monetary values **MUST** be serialized as strings (not floating-point numbers) to prevent IEEE-754 precision loss in browser runtimes.
- **BR-003-10**: Date-time values **MUST** be serialized in ISO-8601 UTC format (`yyyy-MM-dd'T'HH:mm:ss'Z'`).
- **BR-003-11**: Date values **MUST** be serialized in ISO-8601 format (`yyyy-MM-dd`).
- **BR-003-12**: All error responses **MUST** use `application/problem+json` content type following RFC 9457.
- **BR-003-13**: Error response bodies **MUST** include `type`, `title`, `status`, `detail`, and `timestamp` properties.
- **BR-003-14**: Internal contextual details (query text, connection parameters, index keys) **SHALL NOT** be exposed in error responses; they **MUST** be routed to secure system logs only.
- **BR-003-15**: A request referencing a non-existent `dashboardId` **MUST** return HTTP 404.
- **BR-003-16**: A request with invalid filter parameter types or missing required filters **MUST** return HTTP 400.
- **BR-003-17**: Internal execution faults or unsupported column type detection **MUST** return HTTP 500.
- **BR-003-18**: Nullable column values **MUST** be represented as explicit JSON `null`.

## 3. Acceptance Criteria / Scenarios

```gherkin
Scenario: Successful query execution with filters
  Given the dashboard "merchant_settlement_ledger" is configured and cached
  When I POST to "/api/dashboards/merchant_settlement_ledger/query" with offset=0, limit=20, and filter_inputs containing channel_filter="BANK_TRANSFER"
  Then the response status is 200
  And the response content-type is "application/json"
  And the response body contains "dashboard_id" = "merchant_settlement_ledger"
  And the response body contains "items" array with up to 20 rows
  And the response body contains "links.self" with the current pagination state

Scenario: Pagination with next link
  Given the dashboard query returns more than 20 results
  When I POST with offset=0, limit=20
  Then the response contains "links.next" pointing to offset=20
  And the "items" array contains exactly 20 rows

Scenario: Pagination at end of results
  Given the dashboard query returns exactly 15 results for offset=40
  When I POST with offset=40, limit=20
  Then the response does not contain "links.next"
  And the "items" array contains 15 rows

Scenario: Previous link included for non-zero offset
  Given offset=20 is requested
  When the query executes successfully
  Then the response contains "links.prev" pointing to offset=0

Scenario: Non-existent dashboard returns 404
  Given no dashboard configuration exists for "nonexistent_dashboard"
  When I POST to "/api/dashboards/nonexistent_dashboard/query"
  Then the response status is 404
  And the response content-type is "application/problem+json"
  And the response body contains "type", "title", "status", "detail", "timestamp"

Scenario: Invalid filter type returns 400
  Given the dashboard expects filter "start_date" of type date
  When I POST with filter_inputs containing start_date="2026/05/18" (invalid format)
  Then the response status is 400
  And the response content-type is "application/problem+json"
  And the detail field describes the expected ISO-8601 format

Scenario: Missing required filter returns 400
  Given the dashboard defines "channel_filter" as required
  When I POST without providing "channel_filter" in filter_inputs
  Then the response status is 400
  And the response describes the missing required filter

Scenario: Decimal values serialized as strings
  Given the dashboard column "net_amount" is type "decimal"
  When query results contain a BigDecimal value of 12450.75
  Then the JSON response serializes it as the string "12450.75"

Scenario: Internal error does not leak implementation details
  Given a database connection failure occurs
  When the query execution fails internally
  Then the response status is 500
  And the response content-type is "application/problem+json"
  And the detail field contains a generic error message
  And no SQL query text, connection strings, or stack traces are present in the response
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Security | ADR-18 | No internal details in error responses; input validation on all parameters | TBD |
| Logging | ADR-16 | Request/response logging with correlation IDs; errors logged with full context | TBD |
| i18n | ADR-25, ADR-26 | Error messages support localization via message bundles | TBD |
| Observability | ADR-20, ADR-21 | Each query execution traced; spans for parameter binding, execution, masking | TBD |
| Error Handling | ADR-23 | All errors mapped to RFC 9457 Problem Details format | TBD |
| Validation | ADR-24 | Request DTO validated with Bean Validation annotations | TBD |
| API Documentation | ADR-27 | OpenAPI 3.0.3 spec generated for the query endpoint | TBD |

## 5. Related ADRs

- ADR-08: Application Framework (Quarkus) — RESTEasy Reactive for endpoint implementation
- ADR-10: Software Architecture (Hexagonal) — Controller as inbound adapter
- ADR-18: Security Implementation (OWASP) — Input validation, error sanitization
- ADR-20: Observability Strategy — Request tracing and metrics
- ADR-21: Distributed Tracing — OpenTelemetry span creation per query
- ADR-23: REST API Error Handling — RFC 9457 Problem Details format
- ADR-24: Data Transfer & Validation — DTO patterns with Bean Validation
- ADR-25: Internationalization Strategy — Localized error messages
- ADR-27: API Documentation Localization — OpenAPI spec generation

## 6. Scope

### In-Scope

- POST endpoint definition and request/response schemas
- Pagination request parameters (offset, limit)
- Filter input parameter binding
- Hypermedia navigation links (self, prev, next)
- Response data type serialization rules (decimal as string, ISO-8601 dates)
- RFC 9457 error response format for 400, 404, and 500 errors
- Error detail sanitization (no internal leakage)

### Out-of-Scope

- Dashboard YAML configuration format (see BR-002)
- SQL execution engine and type enforcement (see BR-004)
- Look-ahead pagination implementation (see BR-004)
- Authentication and authorization
- Rate limiting

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
          "500": {
            "description": "Internal execution fault or unsupported column type detected.",
            "content": {
              "application/problem+json": {
                "schema": { "$ref": "#/components/schemas/ProblemDetails" }
              }
            }
          }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "QueryRequest": {
        "type": "object",
        "required": ["offset", "limit"],
        "properties": {
          "offset": { "type": "integer", "minimum": 0, "example": 0 },
          "limit": { "type": "integer", "minimum": 1, "maximum": 100, "example": 20 },
          "filter_inputs": { "type": "object", "additionalProperties": true }
        }
      },
      "QueryResponse": {
        "type": "object",
        "required": ["dashboard_id", "links", "items"],
        "properties": {
          "dashboard_id": { "type": "string" },
          "links": { "$ref": "#/components/schemas/HypermediaLinks" },
          "items": { "type": "array", "items": { "$ref": "#/components/schemas/Row" } }
        }
      },
      "HypermediaLinks": {
        "type": "object",
        "required": ["self"],
        "properties": {
          "self": { "type": "string" },
          "prev": { "type": "string" },
          "next": { "type": "string" }
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
        "required": ["type", "title", "status", "detail", "timestamp"],
        "properties": {
          "type": { "type": "string" },
          "title": { "type": "string" },
          "status": { "type": "integer" },
          "detail": { "type": "string" },
          "instance": { "type": "string" },
          "timestamp": { "type": "string", "format": "date-time" }
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
  "type": "https://api.reportingengine.internal/errors/invalid-filter-parameter",
  "title": "Invalid Filter Parameter",
  "status": 400,
  "detail": "The filter 'start_date' expects a valid ISO-8601 calendar date format. Provided value: '2026/05/18'",
  "instance": "/api/dashboards/merchant_settlement_ledger/query",
  "timestamp": "2026-05-18T12:26:04Z"
}
```

**HTTP 500 — Internal Execution Fault:**
```json
{
  "type": "https://api.reportingengine.internal/errors/unsupported-column-type",
  "title": "Unsupported Column Type Error",
  "status": 500,
  "detail": "An internal query mapping fault occurred. Unstructured or complex nested column schemas are blocked for security.",
  "instance": "/api/dashboards/merchant_settlement_ledger/query",
  "timestamp": "2026-05-18T12:26:04Z"
}
```
