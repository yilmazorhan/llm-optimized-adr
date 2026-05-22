# BR-002: Git-Backed Dashboard Definitions

## 1. Summary

As a platform engineer, I want dashboard configurations to be defined as versioned YAML files in a Git repository, so that reporting schemas are auditable, version-controlled, and validated automatically by the backend engine.

This feature defines the YAML specification for dashboard configurations including SQL query templates, column definitions, dynamic filters, sorting attributes, and data masking rules. The Quarkus backend validates these configurations against a JSON meta-schema upon synchronization or startup, ensuring structural correctness before any query execution.

## 2. Business Rules

- **BR-002-01**: Dashboard configurations **MUST** be stored as YAML files in a `dashboards/` directory within the Git repository.
- **BR-002-02**: Each dashboard YAML file **MUST** contain a unique `dashboard_id` matching the pattern `^[a-zA-Z0-9_-]+$`.
- **BR-002-03**: Each dashboard YAML file **MUST** define `dashboard_id`, `name`, `sqltable`, and `filters` as top-level required properties.
- **BR-002-04**: The `sqltable` object **MUST** contain `datasource`, `sql`, and `columns` properties.
- **BR-002-05**: The `sql` property **MUST** use YAML literal block scalar (`|`) for multi-line SQL queries with named parameters (`:parameter_name` syntax).
- **BR-002-06**: Each column definition **MUST** specify `name` and `type` properties; `type` **MUST** be one of: `string`, `integer`, `decimal`, `double`, `boolean`, `date`, `datetime`.
- **BR-002-07**: Column definitions **MAY** include `format` (display pattern) and `label` (human-readable header) properties.
- **BR-002-08**: Filter definitions **MUST** specify `name`, `type`, and `required` properties; `type` **MUST** be one of: `date`, `datetime`, `boolean`, `dynamic_kv`.
- **BR-002-09**: Filters of type `dynamic_kv` **MUST** include a `lookup_sql` property containing the SQL query for key-value pair retrieval.
- **BR-002-10**: Filters of type `dynamic_kv` **MAY** specify a separate `datasource`; if omitted, the `sqltable` datasource **SHALL** be used.
- **BR-002-11**: The `sorting_attributes` section **MAY** be defined as a key-value map of column identifiers to sort parameter names.
- **BR-002-12**: The `masking_rules` section **MAY** be defined per column with a required `strategy` property.
- **BR-002-13**: Masking strategy **MUST** be one of: `partial_text`, `suffix_only`, `email_mask`, `full_redact`.
- **BR-002-14**: The `suffix_only` masking strategy **MAY** include a `visible_chars_end` integer property (minimum 0) specifying how many trailing characters remain visible.
- **BR-002-15**: The Quarkus backend **MUST** validate all dashboard YAML files against the JSON meta-schema on startup and upon Git directory modification.
- **BR-002-16**: Data masking **MUST** be applied in server heap space prior to client data emission; unmasked values **SHALL NOT** be transmitted across network boundaries.

## 3. Acceptance Criteria / Scenarios

```gherkin
Scenario: Valid dashboard YAML is loaded successfully
  Given a YAML file "dashboards/merchant_settlement_ledger.yaml" exists with valid structure
  When the Quarkus backend starts or detects a file modification
  Then the dashboard configuration is parsed and cached in-memory
  And no validation errors are reported

Scenario: Dashboard YAML missing required field is rejected
  Given a YAML file exists without the required "sqltable" property
  When the backend attempts to validate the configuration
  Then a schema validation error is raised
  And the dashboard is not loaded into the cache

Scenario: Invalid column type is rejected
  Given a YAML file defines a column with type "json"
  When the backend validates the configuration
  Then a schema validation error is raised indicating invalid enum value for column type

Scenario: Dynamic filter without lookup_sql is rejected
  Given a YAML file defines a filter of type "dynamic_kv" without a "lookup_sql" property
  When the backend validates the configuration
  Then a schema validation error is raised indicating missing required property

Scenario: Partial text masking applied correctly
  Given a dashboard defines masking_rules with strategy "partial_text" for column "merchant_name"
  When query results contain "Orhan Yılmaz" for that column
  Then the masked value returned to the client is "O**** Y*****"

Scenario: Suffix-only masking applied correctly
  Given a dashboard defines masking_rules with strategy "suffix_only" and visible_chars_end=4 for column "destination_iban"
  When query results contain "TR5600010009" for that column
  Then the masked value returned to the client is "XXXXXXXX0009"

Scenario: Email masking applied correctly
  Given a dashboard defines masking_rules with strategy "email_mask" for column "merchant_email"
  When query results contain "user@paycell.com.tr" for that column
  Then the masked value returned to the client is "u***r@paycell.com.tr"

Scenario: Full redact masking applied correctly
  Given a dashboard defines masking_rules with strategy "full_redact" for a column
  When query results contain any value for that column
  Then the masked value returned to the client is "[REDACTED]"

Scenario: Dashboard ID uniqueness enforced
  Given two YAML files exist with the same "dashboard_id" value
  When the backend loads configurations
  Then a conflict error is reported and the duplicate is rejected
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Security | ADR-18 | Masking applied server-side before serialization; no raw sensitive data transmitted | TBD |
| Logging | ADR-16 | Schema validation errors logged with structured JSON format | TBD |
| i18n | ADR-25, ADR-26 | Column labels support localized display names | TBD |
| Observability | ADR-20, ADR-21 | Schema loading and validation traced via OpenTelemetry spans | TBD |
| Configuration | ADR-15 | Datasource names mapped to configured database connections in application.properties | TBD |
| Error Handling | ADR-23 | Validation failures returned as RFC 9457 Problem Details | TBD |
| Code Quality | ADR-12 | YAML parsing POJOs follow Google Java Style, validated by Spotless | TBD |

## 5. Related ADRs

- ADR-08: Application Framework (Quarkus) — Runtime engine for YAML processing and validation
- ADR-10: Software Architecture (Hexagonal) — Schema registry as infrastructure adapter
- ADR-12: Code Quality Standards — Code formatting and static analysis
- ADR-15: Application Configuration Management — Datasource connection mapping
- ADR-16: Structured Logging Strategy — Validation error logging
- ADR-18: Security Implementation (OWASP) — Data masking as security control
- ADR-23: REST API Error Handling — RFC 9457 error responses for validation failures
- ADR-25: Internationalization Strategy — Localized column labels

## 6. Scope

### In-Scope

- YAML dashboard configuration file format specification
- JSON meta-schema for structural validation
- Column type definitions and format patterns
- Filter type definitions (date, datetime, boolean, dynamic_kv)
- Sorting attribute configuration
- Data masking rule definitions and strategies (partial_text, suffix_only, email_mask, full_redact)
- Git-based file monitoring for configuration changes

### Out-of-Scope

- REST API endpoint implementation (see BR-003)
- SQL query execution and type enforcement (see BR-004)
- Pagination logic (see BR-004)
- User authentication and authorization
- Frontend dashboard rendering

## 7. Dashboard YAML Configuration Example

```yaml
dashboard_id: merchant_settlement_ledger
name: Merchant Settlement and Performance Ledger
sqltable:
  datasource: settlement_db
  sql: |
    SELECT 
      ledger_id,
      merchant_id,
      settlement_date,
      processed_at,
      merchant_name,
      merchant_email,
      destination_iban,
      payout_channel,
      conversion_rate,
      is_settled,
      net_amount
    FROM merchant_settlement_ledger 
    WHERE payout_channel = :channel_filter 
      AND settlement_date >= :start_date 
      AND settlement_date <= :end_date 
      AND is_settled = :settlement_status_filter
    ORDER BY settlement_date DESC
  columns:
    - name: ledger_id
      type: integer
      label: Ledger ID
    - name: merchant_id
      type: integer
      label: Merchant ID
    - name: settlement_date
      type: date
      format: yyyy-MM-dd
      label: Settlement Date
    - name: processed_at
      type: datetime
      format: "yyyy-MM-dd'T'HH:mm:ss'Z'"
      label: Processed At
    - name: merchant_name
      type: string
      label: Merchant Name
    - name: merchant_email
      type: string
      label: Email
    - name: destination_iban
      type: string
      label: IBAN
    - name: payout_channel
      type: string
      label: Payout Channel
    - name: conversion_rate
      type: double
      format: "#,##0.0000"
      label: Conversion Rate
    - name: is_settled
      type: boolean
      label: Settled
    - name: net_amount
      type: decimal
      format: "#,##0.00"
      label: Net Amount

filters:
  - name: channel_filter
    type: dynamic_kv
    required: true
    datasource: settlement_db
    lookup_sql: |
      SELECT DISTINCT channel_code AS key, channel_name AS value 
      FROM payment_channels 
      WHERE is_active = true
  - name: start_date
    type: date
    required: true
  - name: end_date
    type: date
    required: false
  - name: settlement_status_filter
    type: boolean
    required: true

sorting_attributes:
  id: sort_by_id
  settlement_date: sort_by_date
  net_amount: sort_by_amount

masking_rules:
  merchant_name:
    strategy: partial_text
  merchant_email:
    strategy: email_mask
  destination_iban:
    strategy: suffix_only
    visible_chars_end: 4
```

## 8. JSON Validation Meta-Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DashboardYAMLValidationSchema",
  "type": "object",
  "required": ["dashboard_id", "name", "sqltable", "filters"],
  "properties": {
    "dashboard_id": {
      "type": "string",
      "pattern": "^[a-zA-Z0-9_-]+$"
    },
    "name": { "type": "string" },
    "sqltable": {
      "type": "object",
      "required": ["datasource", "sql", "columns"],
      "properties": {
        "datasource": { "type": "string" },
        "sql": { "type": "string" },
        "columns": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["name", "type"],
            "properties": {
              "name": { "type": "string" },
              "type": { "type": "string", "enum": ["string", "integer", "decimal", "double", "boolean", "date", "datetime"] },
              "format": { "type": "string" },
              "label": { "type": "string" }
            }
          }
        }
      }
    },
    "filters": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "type", "required"],
        "properties": {
          "name": { "type": "string" },
          "required": { "type": "boolean" },
          "type": { "type": "string", "enum": ["date", "datetime", "boolean", "dynamic_kv"] },
          "datasource": { "type": "string" },
          "lookup_sql": { "type": "string" }
        },
        "if": { "properties": { "type": { "const": "dynamic_kv" } } },
        "then": { "required": ["lookup_sql"] }
      }
    },
    "sorting_attributes": {
      "type": "object",
      "additionalProperties": { "type": "string" }
    },
    "masking_rules": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "required": ["strategy"],
        "properties": {
          "strategy": { "type": "string", "enum": ["partial_text", "suffix_only", "email_mask", "full_redact"] },
          "visible_chars_end": { "type": "integer", "minimum": 0 }
        }
      }
    }
  }
}
```
