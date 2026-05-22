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

```gherkin
Scenario: HSQLDB starts in Oracle compatibility mode
  Given the Quarkus test profile is active
  When the test datasource is initialized
  Then HSQLDB accepts Oracle SQL syntax (NUMBER, VARCHAR2, SYSDATE, NVL)
  And the JDBC connection URL contains "sql.syntax_ora=true"

Scenario: Sample schema is created on test startup
  Given test initialization scripts exist in src/test/resources/db/
  When the test suite starts
  Then the merchant_settlement_ledger table exists with all expected columns
  And the payment_channels lookup table exists with active/inactive entries

Scenario: Pagination validation with 50+ rows
  Given the merchant_settlement_ledger table contains 55 seed rows
  When a query requests offset=0, limit=20
  Then 21 rows are fetched (look-ahead)
  And 20 rows are returned to the caller
  And hasNext is true

Scenario: End of results detected correctly
  Given the merchant_settlement_ledger table contains 55 seed rows
  When a query requests offset=40, limit=20
  Then 15 rows are returned
  And hasNext is false

Scenario: Null values present in test data
  Given the merchant_settlement_ledger table has nullable columns (end_date, conversion_rate)
  When a query returns rows with NULL values
  Then the JSON serialization outputs explicit null for those fields

Scenario: BigDecimal precision preserved in test data
  Given rows contain net_amount values: 0, 12450.75, 99999.9999, 0.00000001
  When these values are read via JDBC
  Then java.math.BigDecimal preserves exact precision without floating-point artifacts

Scenario: Date range filter works against HSQLDB
  Given rows exist with settlement_date spanning 2025-01-01 to 2026-12-31
  When a query filters with start_date=2026-01-01 and end_date=2026-06-30
  Then only rows within that date range are returned

Scenario: Boolean filter works against HSQLDB
  Given rows exist with is_settled values of both 1 (true) and 0 (false)
  When a query filters with settlement_status_filter=true
  Then only rows where is_settled=1 are returned

Scenario: Dynamic KV lookup returns active channels only
  Given the payment_channels table has 5 active and 2 inactive entries
  When the dynamic_kv filter executes its lookup_sql
  Then only 5 active channel key-value pairs are returned

Scenario: Type enforcement test table triggers security abort
  Given the type_enforcement_test table contains a CLOB column
  When the type enforcement guard inspects ResultSetMetaData
  Then the query execution aborts with a security violation

Scenario: No Docker required for test execution
  Given Docker is not running on the developer machine
  When I run "./mvnw test"
  Then all tests pass using in-memory HSQLDB
  And no ContainerLaunchException or DockerNotAvailableException occurs
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Configuration | ADR-15 | HSQLDB datasource configured in %test profile of application.properties | TBD |
| Architecture | ADR-10 | Test datasource injected via same port interface used in production | TBD |
| Data Migration | ADR-14 | Test scripts use Flyway-compatible naming for schema init | TBD |
| Code Quality | ADR-12 | Test classes follow Google Java Style, validated by Spotless | TBD |
| Logging | ADR-16 | Test execution logs SQL statements at DEBUG level | TBD |
| Security | ADR-18 | Type enforcement guard validated against HSQLDB metadata | TBD |
| Build | ADR-03 | HSQLDB dependency scoped to `test` in Maven POM | TBD |

## 5. Related ADRs

- ADR-03: Build Automation — HSQLDB test dependency declaration in parent POM
- ADR-07: Containerized Development Environment — HSQLDB as Docker-free alternative for unit tests
- ADR-10: Software Architecture (Hexagonal) — Datasource adapter swapped for HSQLDB in test
- ADR-14: Data Migration Strategy — Test schema scripts follow Flyway naming patterns
- ADR-15: Application Configuration Management — %test profile datasource configuration
- ADR-23: REST API Error Handling — Error responses validated against in-memory data
- ADR-24: Data Transfer & Validation — DTO serialization validated with known test data

## 6. Scope

### In-Scope

- HSQLDB dependency and Quarkus %test profile configuration
- Oracle-compatible sample table schemas (DDL scripts)
- Sample data population scripts (DML with 55+ rows)
- Lookup table for dynamic_kv filter testing
- Type enforcement guard test table (disallowed types)
- Null value, precision, and boundary test data
- Named datasource configuration matching dashboard YAML references

### Out-of-Scope

- Production database setup or connectivity
- Docker-based Testcontainers (covered by ADR-07 for integration tests)
- Performance/load testing data volumes
- Schema migration versioning for production (see ADR-14)
- Dashboard YAML configuration format (see BR-002)

## 7. Configuration

### Maven Dependency (Parent POM)

```xml
<properties>
    <hsqldb.version>2.7.4</hsqldb.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.hsqldb</groupId>
            <artifactId>hsqldb</artifactId>
            <version>${hsqldb.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Quarkus Test Profile Configuration (`application.properties`)

```properties
# =============================================================================
# TEST PROFILE — HSQLDB In-Memory (Oracle Compatibility)
# =============================================================================
%test.quarkus.datasource.settlement_db.db-kind=other
%test.quarkus.datasource.settlement_db.jdbc.driver=org.hsqldb.jdbc.JDBCDriver
%test.quarkus.datasource.settlement_db.jdbc.url=jdbc:hsqldb:mem:testdb;sql.syntax_ora=true
%test.quarkus.datasource.settlement_db.username=SA
%test.quarkus.datasource.settlement_db.password=
%test.quarkus.datasource.settlement_db.jdbc.min-size=1
%test.quarkus.datasource.settlement_db.jdbc.max-size=5

# Schema initialization
%test.quarkus.flyway.settlement_db.migrate-at-start=true
%test.quarkus.flyway.settlement_db.locations=db/test-migration
%test.quarkus.flyway.settlement_db.baseline-on-migrate=true
```

## 8. Sample Table Schemas

### DDL Script: `src/test/resources/db/test-migration/V1.0.0__create_test_schemas.sql`

```sql
-- =============================================================================
-- MERCHANT SETTLEMENT LEDGER — Primary test table for query execution
-- Oracle-compatible syntax (HSQLDB sql.syntax_ora=true)
-- =============================================================================

CREATE TABLE merchant_settlement_ledger (
    ledger_id        NUMBER(19)      NOT NULL,
    merchant_id      NUMBER(19)      NOT NULL,
    settlement_date  DATE            NOT NULL,
    processed_at     TIMESTAMP       NOT NULL,
    merchant_name    VARCHAR2(200)   NOT NULL,
    merchant_email   VARCHAR2(200)   NOT NULL,
    destination_iban VARCHAR2(34)    NOT NULL,
    payout_channel   VARCHAR2(50)    NOT NULL,
    conversion_rate  FLOAT           NULL,
    is_settled       NUMBER(1)       NOT NULL,
    net_amount       NUMBER(19,4)    NOT NULL,
    CONSTRAINT pk_settlement_ledger PRIMARY KEY (ledger_id)
);

CREATE INDEX idx_ledger_channel ON merchant_settlement_ledger (payout_channel);
CREATE INDEX idx_ledger_date ON merchant_settlement_ledger (settlement_date);
CREATE INDEX idx_ledger_settled ON merchant_settlement_ledger (is_settled);

-- =============================================================================
-- PAYMENT CHANNELS — Lookup table for dynamic_kv filter validation
-- =============================================================================

CREATE TABLE payment_channels (
    channel_code     VARCHAR2(30)    NOT NULL,
    channel_name     VARCHAR2(100)   NOT NULL,
    is_active        NUMBER(1)       NOT NULL DEFAULT 1,
    CONSTRAINT pk_payment_channels PRIMARY KEY (channel_code)
);

-- =============================================================================
-- TYPE ENFORCEMENT TEST TABLE — Contains disallowed column types
-- Used to validate security abort mechanism
-- =============================================================================

CREATE TABLE type_enforcement_test (
    id               NUMBER(19)      NOT NULL,
    normal_col       VARCHAR2(100)   NOT NULL,
    clob_col         CLOB            NULL,
    blob_col         BLOB            NULL,
    CONSTRAINT pk_type_enforcement PRIMARY KEY (id)
);
```

## 9. Sample Data

### DML Script: `src/test/resources/db/test-migration/V1.0.1__insert_test_data.sql`

```sql
-- =============================================================================
-- PAYMENT CHANNELS — Lookup data (5 active, 2 inactive)
-- =============================================================================

INSERT INTO payment_channels (channel_code, channel_name, is_active) VALUES ('BANK_TRANSFER', 'Bank Wire Transfer', 1);
INSERT INTO payment_channels (channel_code, channel_name, is_active) VALUES ('CREDIT_CARD', 'Credit Card Payment', 1);
INSERT INTO payment_channels (channel_code, channel_name, is_active) VALUES ('MOBILE_WALLET', 'Mobile Wallet', 1);
INSERT INTO payment_channels (channel_code, channel_name, is_active) VALUES ('CRYPTO', 'Cryptocurrency', 1);
INSERT INTO payment_channels (channel_code, channel_name, is_active) VALUES ('DIRECT_DEBIT', 'Direct Debit', 1);
INSERT INTO payment_channels (channel_code, channel_name, is_active) VALUES ('CHEQUE', 'Paper Cheque', 0);
INSERT INTO payment_channels (channel_code, channel_name, is_active) VALUES ('CASH', 'Cash Pickup', 0);

-- =============================================================================
-- MERCHANT SETTLEMENT LEDGER — 55 rows for pagination testing
-- Covers: all channel types, date ranges, null values, precision edge cases
-- =============================================================================

-- Rows 1-10: BANK_TRANSFER, settled, 2026 dates
INSERT INTO merchant_settlement_ledger VALUES (1, 1001, DATE '2026-01-15', TIMESTAMP '2026-01-15 10:30:00', 'Acme Corp', 'finance@acme.com', 'TR330006100519786457841326', 'BANK_TRANSFER', 1.0000, 1, 15000.7500);
INSERT INTO merchant_settlement_ledger VALUES (2, 1002, DATE '2026-01-20', TIMESTAMP '2026-01-20 14:15:00', 'Beta Industries', 'pay@beta.io', 'DE89370400440532013000', 'BANK_TRANSFER', 1.0850, 1, 28750.5000);
INSERT INTO merchant_settlement_ledger VALUES (3, 1003, DATE '2026-02-01', TIMESTAMP '2026-02-01 09:00:00', 'Gamma Solutions', 'billing@gamma.dev', 'GB29NWBK60161331926819', 'BANK_TRANSFER', 0.8500, 1, 5200.0000);
INSERT INTO merchant_settlement_ledger VALUES (4, 1004, DATE '2026-02-14', TIMESTAMP '2026-02-14 16:45:00', 'Delta Payments', 'ap@delta.co', 'FR7630006000011234567890189', 'BANK_TRANSFER', 1.1200, 1, 99999.9999);
INSERT INTO merchant_settlement_ledger VALUES (5, 1005, DATE '2026-03-01', TIMESTAMP '2026-03-01 08:00:00', 'Epsilon Ltd', 'invoices@epsilon.uk', 'GB82WEST12345698765432', 'BANK_TRANSFER', NULL, 1, 750.2500);
INSERT INTO merchant_settlement_ledger VALUES (6, 1006, DATE '2026-03-15', TIMESTAMP '2026-03-15 11:30:00', 'Zeta Holdings', 'treasury@zeta.com', 'TR120006200000000001234567', 'BANK_TRANSFER', 1.0000, 1, 42000.0000);
INSERT INTO merchant_settlement_ledger VALUES (7, 1007, DATE '2026-04-01', TIMESTAMP '2026-04-01 13:00:00', 'Eta Services', 'billing@eta.net', 'NL91ABNA0417164300', 'BANK_TRANSFER', 1.3250, 1, 18500.8800);
INSERT INTO merchant_settlement_ledger VALUES (8, 1008, DATE '2026-04-10', TIMESTAMP '2026-04-10 15:20:00', 'Theta Corp', 'finance@theta.org', 'ES9121000418450200051332', 'BANK_TRANSFER', 1.1000, 1, 0.0100);
INSERT INTO merchant_settlement_ledger VALUES (9, 1009, DATE '2026-05-01', TIMESTAMP '2026-05-01 07:45:00', 'Iota Group', 'pay@iota.biz', 'IT60X0542811101000000123456', 'BANK_TRANSFER', NULL, 1, 67890.1234);
INSERT INTO merchant_settlement_ledger VALUES (10, 1010, DATE '2026-05-15', TIMESTAMP '2026-05-15 12:00:00', 'Kappa Digital', 'ap@kappa.tech', 'TR250006200000000009876543', 'BANK_TRANSFER', 1.0000, 1, 0.00000000);

-- Rows 11-20: CREDIT_CARD, mixed settled/unsettled
INSERT INTO merchant_settlement_ledger VALUES (11, 1011, DATE '2026-01-05', TIMESTAMP '2026-01-05 09:30:00', 'Lambda Shop', 'orders@lambda.store', 'TR110006100519786457841111', 'CREDIT_CARD', 1.0000, 1, 320.5000);
INSERT INTO merchant_settlement_ledger VALUES (12, 1012, DATE '2026-01-18', TIMESTAMP '2026-01-18 10:00:00', 'Mu Retail', 'finance@mu.shop', 'DE44370400440532019999', 'CREDIT_CARD', 1.0850, 0, 1580.7500);
INSERT INTO merchant_settlement_ledger VALUES (13, 1013, DATE '2026-02-10', TIMESTAMP '2026-02-10 14:30:00', 'Nu Electronics', 'pay@nu.tech', 'GB55NWBK60161331920001', 'CREDIT_CARD', 0.9200, 1, 4750.0000);
INSERT INTO merchant_settlement_ledger VALUES (14, 1014, DATE '2026-02-28', TIMESTAMP '2026-02-28 16:00:00', 'Xi Market', 'billing@xi.market', 'FR7630006000011234500000001', 'CREDIT_CARD', NULL, 0, 890.2500);
INSERT INTO merchant_settlement_ledger VALUES (15, 1015, DATE '2026-03-12', TIMESTAMP '2026-03-12 08:15:00', 'Omicron Foods', 'orders@omicron.food', 'TR990006200000000005551234', 'CREDIT_CARD', 1.0000, 1, 12400.0000);
INSERT INTO merchant_settlement_ledger VALUES (16, 1016, DATE '2026-03-25', TIMESTAMP '2026-03-25 11:45:00', 'Pi Fashion', 'ap@pi.style', 'NL02ABNA0417164301', 'CREDIT_CARD', 1.2100, 0, 3350.6000);
INSERT INTO merchant_settlement_ledger VALUES (17, 1017, DATE '2026-04-05', TIMESTAMP '2026-04-05 13:30:00', 'Rho Travel', 'payments@rho.travel', 'ES8021000418450200099887', 'CREDIT_CARD', 1.0500, 1, 8900.0000);
INSERT INTO merchant_settlement_ledger VALUES (18, 1018, DATE '2026-04-20', TIMESTAMP '2026-04-20 15:00:00', 'Sigma Health', 'billing@sigma.health', 'IT28X0542811101000000654321', 'CREDIT_CARD', 1.0000, 1, 22100.5500);
INSERT INTO merchant_settlement_ledger VALUES (19, 1019, DATE '2026-05-08', TIMESTAMP '2026-05-08 07:30:00', 'Tau Books', 'finance@tau.read', 'GB12WEST12345698760001', 'CREDIT_CARD', NULL, 0, 450.0000);
INSERT INTO merchant_settlement_ledger VALUES (20, 1020, DATE '2026-05-22', TIMESTAMP '2026-05-22 12:30:00', 'Upsilon Games', 'pay@upsilon.game', 'DE77370400440532013555', 'CREDIT_CARD', 0.9800, 1, 6780.9900);

-- Rows 21-30: MOBILE_WALLET, all settled
INSERT INTO merchant_settlement_ledger VALUES (21, 1021, DATE '2026-01-08', TIMESTAMP '2026-01-08 10:00:00', 'Phi Mobile', 'wallet@phi.app', 'TR550006100519786457849999', 'MOBILE_WALLET', 1.0000, 1, 150.0000);
INSERT INTO merchant_settlement_ledger VALUES (22, 1022, DATE '2026-01-22', TIMESTAMP '2026-01-22 11:30:00', 'Chi Services', 'pay@chi.svc', 'FR7630006000011234500000099', 'MOBILE_WALLET', 1.0000, 1, 275.5000);
INSERT INTO merchant_settlement_ledger VALUES (23, 1023, DATE '2026-02-05', TIMESTAMP '2026-02-05 14:00:00', 'Psi Delivery', 'billing@psi.deliver', 'NL55ABNA0417164399', 'MOBILE_WALLET', 1.0000, 1, 89.9900);
INSERT INTO merchant_settlement_ledger VALUES (24, 1024, DATE '2026-02-19', TIMESTAMP '2026-02-19 09:45:00', 'Omega Rides', 'finance@omega.ride', 'ES1121000418450200011223', 'MOBILE_WALLET', 1.0000, 1, 1200.0000);
INSERT INTO merchant_settlement_ledger VALUES (25, 1025, DATE '2026-03-03', TIMESTAMP '2026-03-03 16:15:00', 'Alpha2 Subscriptions', 'subs@alpha2.io', 'IT99X0542811101000000111222', 'MOBILE_WALLET', 1.0000, 1, 49.9900);
INSERT INTO merchant_settlement_ledger VALUES (26, 1026, DATE '2026-03-18', TIMESTAMP '2026-03-18 08:30:00', 'Beta2 Cloud', 'cloud@beta2.dev', 'GB33WEST12345698760099', 'MOBILE_WALLET', 1.0000, 1, 599.0000);
INSERT INTO merchant_settlement_ledger VALUES (27, 1027, DATE '2026-04-02', TIMESTAMP '2026-04-02 10:45:00', 'Gamma2 Streaming', 'pay@gamma2.stream', 'DE11370400440532013888', 'MOBILE_WALLET', 1.0000, 1, 14.9900);
INSERT INTO merchant_settlement_ledger VALUES (28, 1028, DATE '2026-04-16', TIMESTAMP '2026-04-16 13:15:00', 'Delta2 Fitness', 'billing@delta2.fit', 'TR880006200000000003337777', 'MOBILE_WALLET', 1.0000, 1, 79.0000);
INSERT INTO merchant_settlement_ledger VALUES (29, 1029, DATE '2026-05-05', TIMESTAMP '2026-05-05 15:30:00', 'Epsilon2 Music', 'royalties@eps2.music', 'FR7630006000011234500000055', 'MOBILE_WALLET', 1.0000, 1, 2400.0000);
INSERT INTO merchant_settlement_ledger VALUES (30, 1030, DATE '2026-05-20', TIMESTAMP '2026-05-20 07:00:00', 'Zeta2 Parking', 'ops@zeta2.park', 'NL77ABNA0417164355', 'MOBILE_WALLET', 1.0000, 1, 35.0000);

-- Rows 31-40: CRYPTO, mixed, includes edge-case precision values
INSERT INTO merchant_settlement_ledger VALUES (31, 1031, DATE '2026-01-10', TIMESTAMP '2026-01-10 12:00:00', 'Eta2 Exchange', 'settlements@eta2.ex', 'TR220006100519786457840001', 'CRYPTO', 62145.12340000, 1, 0.00000001);
INSERT INTO merchant_settlement_ledger VALUES (32, 1032, DATE '2026-01-25', TIMESTAMP '2026-01-25 14:30:00', 'Theta2 Mining', 'payout@theta2.mine', 'DE22370400440532013777', 'CRYPTO', 62000.0000, 0, 1.50000000);
INSERT INTO merchant_settlement_ledger VALUES (33, 1033, DATE '2026-02-08', TIMESTAMP '2026-02-08 09:15:00', 'Iota2 DeFi', 'treasury@iota2.defi', 'GB88NWBK60161331920088', 'CRYPTO', NULL, 1, 25000.0000);
INSERT INTO merchant_settlement_ledger VALUES (34, 1034, DATE '2026-02-22', TIMESTAMP '2026-02-22 16:30:00', 'Kappa2 NFT', 'sales@kappa2.nft', 'ES5521000418450200077665', 'CRYPTO', 0.0001, 0, 500000.0000);
INSERT INTO merchant_settlement_ledger VALUES (35, 1035, DATE '2026-03-07', TIMESTAMP '2026-03-07 08:45:00', 'Lambda2 Staking', 'rewards@lambda2.stake', 'IT11X0542811101000000999888', 'CRYPTO', 62500.5555, 1, 0.12345678);
INSERT INTO merchant_settlement_ledger VALUES (36, 1036, DATE '2026-03-20', TIMESTAMP '2026-03-20 11:00:00', 'Mu2 Wallet', 'payouts@mu2.wallet', 'FR7630006000011234500000077', 'CRYPTO', 61000.0000, 1, 3.14159265);
INSERT INTO merchant_settlement_ledger VALUES (37, 1037, DATE '2026-04-03', TIMESTAMP '2026-04-03 14:45:00', 'Nu2 Protocol', 'settlements@nu2.proto', 'NL33ABNA0417164377', 'CRYPTO', NULL, 0, 100.0000);
INSERT INTO merchant_settlement_ledger VALUES (38, 1038, DATE '2026-04-18', TIMESTAMP '2026-04-18 10:30:00', 'Xi2 Labs', 'finance@xi2.labs', 'GB44WEST12345698760077', 'CRYPTO', 63000.9900, 1, 75000.0000);
INSERT INTO merchant_settlement_ledger VALUES (39, 1039, DATE '2026-05-02', TIMESTAMP '2026-05-02 13:00:00', 'Omicron2 Chain', 'ops@omicron2.chain', 'DE55370400440532013222', 'CRYPTO', 62200.0000, 1, 0.5000);
INSERT INTO merchant_settlement_ledger VALUES (40, 1040, DATE '2026-05-18', TIMESTAMP '2026-05-18 15:15:00', 'Pi2 Token', 'treasury@pi2.token', 'TR660006200000000008882222', 'CRYPTO', 61500.0000, 0, 9999.9999);

-- Rows 41-50: DIRECT_DEBIT, settled, includes 2025 dates for range filtering
INSERT INTO merchant_settlement_ledger VALUES (41, 1041, DATE '2025-06-01', TIMESTAMP '2025-06-01 09:00:00', 'Rho2 Insurance', 'claims@rho2.insure', 'TR440006100519786457843333', 'DIRECT_DEBIT', 1.0000, 1, 2500.0000);
INSERT INTO merchant_settlement_ledger VALUES (42, 1042, DATE '2025-07-15', TIMESTAMP '2025-07-15 10:30:00', 'Sigma2 Utilities', 'billing@sigma2.util', 'DE99370400440532013444', 'DIRECT_DEBIT', 1.0000, 1, 180.5000);
INSERT INTO merchant_settlement_ledger VALUES (43, 1043, DATE '2025-09-01', TIMESTAMP '2025-09-01 14:00:00', 'Tau2 Telecom', 'invoices@tau2.tel', 'GB66NWBK60161331920066', 'DIRECT_DEBIT', 1.0000, 1, 89.9900);
INSERT INTO merchant_settlement_ledger VALUES (44, 1044, DATE '2025-11-10', TIMESTAMP '2025-11-10 11:15:00', 'Upsilon2 Energy', 'pay@upsilon2.energy', 'ES3321000418450200055443', 'DIRECT_DEBIT', 1.0000, 1, 320.0000);
INSERT INTO merchant_settlement_ledger VALUES (45, 1045, DATE '2025-12-20', TIMESTAMP '2025-12-20 16:45:00', 'Phi2 Water', 'billing@phi2.water', 'IT55X0542811101000000777666', 'DIRECT_DEBIT', 1.0000, 1, 45.7500);
INSERT INTO merchant_settlement_ledger VALUES (46, 1046, DATE '2026-01-02', TIMESTAMP '2026-01-02 08:00:00', 'Chi2 Internet', 'support@chi2.net', 'FR7630006000011234500000033', 'DIRECT_DEBIT', 1.0000, 1, 69.9900);
INSERT INTO merchant_settlement_ledger VALUES (47, 1047, DATE '2026-02-15', TIMESTAMP '2026-02-15 09:30:00', 'Psi2 Hosting', 'billing@psi2.host', 'NL88ABNA0417164388', 'DIRECT_DEBIT', 1.0000, 1, 199.0000);
INSERT INTO merchant_settlement_ledger VALUES (48, 1048, DATE '2026-03-28', TIMESTAMP '2026-03-28 12:45:00', 'Omega2 SaaS', 'subscriptions@omega2.saas', 'GB22WEST12345698760055', 'DIRECT_DEBIT', 1.0000, 1, 499.0000);
INSERT INTO merchant_settlement_ledger VALUES (49, 1049, DATE '2026-04-30', TIMESTAMP '2026-04-30 14:00:00', 'Alpha3 CRM', 'license@alpha3.crm', 'DE33370400440532013111', 'DIRECT_DEBIT', 1.0000, 1, 1200.0000);
INSERT INTO merchant_settlement_ledger VALUES (50, 1050, DATE '2026-05-10', TIMESTAMP '2026-05-10 10:15:00', 'Beta3 ERP', 'billing@beta3.erp', 'TR770006200000000001119999', 'DIRECT_DEBIT', 1.0000, 1, 3500.0000);

-- Rows 51-55: Mixed channels, boundary values and special cases
INSERT INTO merchant_settlement_ledger VALUES (51, 1051, DATE '2026-06-01', TIMESTAMP '2026-06-01 00:00:00', 'Gamma3 Future', 'future@gamma3.io', 'ES9921000418450200099112', 'BANK_TRANSFER', 1.0000, 0, 10000.0000);
INSERT INTO merchant_settlement_ledger VALUES (52, 1052, DATE '2026-12-31', TIMESTAMP '2026-12-31 23:59:59', 'Delta3 YearEnd', 'eoy@delta3.fin', 'IT22X0542811101000000333444', 'CREDIT_CARD', 1.0000, 1, 55555.5555);
INSERT INTO merchant_settlement_ledger VALUES (53, 1053, DATE '2025-01-01', TIMESTAMP '2025-01-01 00:00:01', 'Epsilon3 OldDate', 'archive@eps3.old', 'FR7630006000011234500000011', 'DIRECT_DEBIT', NULL, 1, 1.0000);
INSERT INTO merchant_settlement_ledger VALUES (54, 1054, DATE '2026-05-22', TIMESTAMP '2026-05-22 12:00:00', 'Zeta3 Today', 'today@zeta3.now', 'NL44ABNA0417164344', 'MOBILE_WALLET', 1.0000, 1, 250.0000);
INSERT INTO merchant_settlement_ledger VALUES (55, 1055, DATE '2026-05-22', TIMESTAMP '2026-05-22 12:00:00', 'Eta3 Duplicate Date', 'dup@eta3.test', 'GB99WEST12345698760033', 'CRYPTO', 62000.0000, 1, 7777.7777);

-- =============================================================================
-- TYPE ENFORCEMENT TEST TABLE — Data for security guard validation
-- =============================================================================

INSERT INTO type_enforcement_test VALUES (1, 'Normal text value', 'This is a CLOB column that should trigger abort', NULL);
INSERT INTO type_enforcement_test VALUES (2, 'Another normal value', NULL, CAST(X'48454C4C4F' AS BLOB));
```

## 10. Test Data Summary

| Aspect | Coverage |
|--------|----------|
| Total rows (merchant_settlement_ledger) | 55 |
| Channels represented | BANK_TRANSFER (11), CREDIT_CARD (10), MOBILE_WALLET (10), CRYPTO (10), DIRECT_DEBIT (10), mixed (4) |
| Settled (is_settled=1) | 42 rows |
| Unsettled (is_settled=0) | 13 rows |
| NULL conversion_rate values | 8 rows (IDs: 5, 9, 14, 19, 33, 37, 53) |
| Date range | 2025-01-01 to 2026-12-31 |
| Precision edge cases | 0.00000001, 0.12345678, 3.14159265, 99999.9999 |
| Zero value | ID 10 (net_amount = 0.00000000) |
| Minimum positive | ID 8 (net_amount = 0.0100) |
| Lookup table active entries | 5 (BANK_TRANSFER, CREDIT_CARD, MOBILE_WALLET, CRYPTO, DIRECT_DEBIT) |
| Lookup table inactive entries | 2 (CHEQUE, CASH) |
| Type enforcement rows | 2 (one with CLOB, one with BLOB) |
