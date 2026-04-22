# BR-000: Platform Operational Foundation

## 1. Summary

As a **development team**, I want **a production-grade Quarkus application skeleton with all cross-cutting operational capabilities pre-configured**, so that **future business features can be built on a fully observable, secure, testable, and deployable foundation without re-implementing infrastructure concerns**.

This BRD establishes the non-business application base delivering: reproducible builds, multi-module Maven architecture with hexagonal package structure, structured JSON logging with correlation IDs, OpenTelemetry distributed tracing with Jaeger, Prometheus metrics with Grafana dashboards, internationalization resource bundle scaffolding, OWASP security headers, rate limiting placeholders, ClickHouse database connectivity foundation, Flyway migration scaffolding, code quality toolchain (Spotless, Checkstyle, PMD, SpotBugs, OWASP Dependency Check), Testcontainers integration test infrastructure, RFC 9457 error handling foundation, audit logging stub, Docker containerization with compose-based local development stack, and automated validation scripts. No business domain logic, entities, or feature endpoints SHALL be included.

## 2. Scope

### In-Scope
- Multi-module Maven parent project with single-responsibility libraries
- Base modules: `platform-api` (placeholder), `platform-infrastructure`, `exception-library`, `i18n-library`, `container-app`
- Libraries MUST have single responsibility; `shared-library` is FORBIDDEN (ADR-09)
- JDK 21.0.4 (Eclipse Temurin) pinned via `.sdkmanrc` (ADR-02)
- Maven Wrapper 3.9.8 (ADR-03)
- Quarkus 3.32.2 BOM with RESTEasy Reactive, Mutiny, OpenTelemetry, Micrometer, SmallRye Health, SmallRye OpenAPI, Hibernate Validator (ADR-08)
- Hexagonal package structure: `domain/`, `application/`, `infrastructure/` with ArchUnit enforcement (ADR-10, ADR-11)
- Code quality toolchain: Spotless (Google Java Format), Checkstyle (Google Checks), PMD (custom ruleset), SpotBugs with FindSecBugs (ADR-12)
- Configuration template (`application.properties`) with `%dev`, `%test`, `%prod` profiles and `${ENV_VAR:default}` substitution (ADR-15)
- Structured JSON logging (prod), plaintext (dev), correlation ID / traceId / spanId MDC fields, redaction utility placeholder (ADR-16)
- Exception handling foundation: `DomainException`, `InfrastructureException` base classes, RFC 9457 `ProblemDetail` mapper stub, error code registry (ADR-17, ADR-23)
- OWASP security headers (`X-Content-Type-Options`, `X-Frame-Options`, `X-XSS-Protection`, `Referrer-Policy`, `Content-Security-Policy`, `Strict-Transport-Security`), rate limiting config placeholders, JWT key generation scaffolding (ADR-18)
- Audit logging: dedicated `AuditLogger` category, separate file handler configuration, `AuditLog` domain model stub (ADR-19)
- OpenTelemetry tracing with Jaeger backend (OTLP gRPC port 4317), W3C Trace Context propagation, sampling config (ADR-21)
- Micrometer Prometheus metrics at `/q/metrics`, SmallRye Health at `/q/health`, custom sample metric bean, Jaeger span metrics exported to Prometheus (ADR-20, ADR-22)
- Grafana dashboards: application overview, framework metrics, provisioned datasources with explicit UIDs (ADR-22)
- ClickHouse 24.x configured in Docker Compose with Tabix Web UI, client configuration class, connection properties (ADR-13)
- Flyway dependency included, migration directory scaffolded, disabled in prod until schemas exist (ADR-14)
- Internationalization: resource bundle directories (`i18n/`), `MessageKeys` constants class (Validation, Log, Api, Entity, Domain categories), `LocaleContextProvider`, `MessageResolver` stubs, supported locales `en,es,fr,de` (ADR-25, ADR-26)
- API documentation localization: `ApiDocumentationService` stub, `/api-info` placeholder endpoint (ADR-27)
- Bean Validation 3.0 with internationalized message keys; sample DTO placeholder (ADR-24)
- Testcontainers: dependencies added (`testcontainers`, `clickhouse`, `junit-jupiter`), `ClickHouseTestResource` stub (ADR-28)
- Code generation templates: documented in ADR-29, referenced for future business module creation
- Docker artifacts: multi-stage `Dockerfile`, `.dockerignore`, non-root user (ADR-05, ADR-31)
- Local development stack via `docker-compose`: ClickHouse, Tabix, Jaeger (with span metrics), Prometheus, Grafana — with health checks, custom networks, volumes (ADR-05, ADR-07)
- Operational diagnostics endpoint at `/internal/diagnostics`
- Automation: `setup.sh` (prerequisite validation), `Makefile` (standardized commands with Docker Compose auto-detection), validation scripts (ADR-04, ADR-06)
- Production build artifacts pipeline: application Docker image, Flyway migration image, Grafana dashboard JSON, error inventory document, OpenAPI specification (ADR-31)

### Explicit Out-of-Scope
- No business entities, domain services, repositories, or domain logic
- No business REST feature endpoints beyond `/internal/diagnostics` and Quarkus defaults (`/q/health`, `/q/metrics`, `/q/openapi`)
- No persistence schema migrations (Flyway placeholder only)
- No user registration, authentication flows, or CRUD endpoints
- No UI assets
- No GraalVM native image build (deferred until business modules added)

## 3. Business Rules

- **BR-000-01**: The project MUST use JDK 21.0.4 (Eclipse Temurin) pinned in `.sdkmanrc`, verified by `java -version` output containing `21.0.4`. (ADR-02)
- **BR-000-02**: The project MUST use Maven Wrapper 3.9.8; all build commands MUST use `./mvnw`. System-wide Maven installations MUST NOT be used for project builds. (ADR-03)
- **BR-000-03**: The project MUST pin all tool versions (Java, Maven, Docker 24.0+, Docker Compose 2.24+) with automated validation via `setup.sh`. Version ranges MUST NOT be used. (ADR-04)
- **BR-000-04**: Docker Engine 24.0+ and Docker Compose 2.24+ MUST be the containerization platform. Scripts MUST support both `docker compose` (plugin) and `docker-compose` (standalone) command variants. (ADR-05)
- **BR-000-05**: A `Makefile` MUST provide standardized commands (setup, dev, build, test, package, start, stop, logs, status, verify). A `setup.sh` MUST validate all prerequisites. (ADR-06)
- **BR-000-06**: Docker Compose MUST manage ClickHouse, Tabix, Jaeger, Prometheus, and Grafana with health checks, custom networks, persistent volumes, and resource limits. (ADR-07)
- **BR-000-07**: Quarkus 3.32.2 BOM MUST be imported. RESTEasy Reactive (`quarkus-rest`) MUST be used. Classic JAX-RS (`quarkus-resteasy`) MUST NOT be used. (ADR-08)
- **BR-000-08**: The project MUST use multi-module Maven architecture with strict separation: Business, External, Web API, Library, and Container module types. Libraries MUST have single responsibility. Names `shared-library`, `common-library`, `utils-library`, `core-library`, `base-library` are FORBIDDEN. (ADR-09)
- **BR-000-09**: Hexagonal architecture (Ports and Adapters) MUST be enforced. Domain MUST NOT depend on infrastructure or application packages. ArchUnit tests MUST fail the build on violations. (ADR-10)
- **BR-000-10**: Package structure MUST follow `com.copilot.quarkus.[layer].[bounded-context].[component]` with `domain/`, `application/`, `infrastructure/` layers. Package names MUST be lowercase with no underscores. (ADR-11)
- **BR-000-11**: Code quality plugins (Spotless, Checkstyle, PMD, SpotBugs with FindSecBugs) MUST be configured and MUST fail the build on violations. Tests MUST NOT be skipped in `./mvnw clean verify`. (ADR-12)
- **BR-000-12**: ClickHouse 24.x MUST be the primary analytical database, configured in Docker Compose with HTTP (8123) and native TCP (9000) ports. Tabix Web UI MUST be enabled for development. (ADR-13)
- **BR-000-13**: Flyway MUST be the schema migration tool. Migration directory (`db/migration`) MUST exist. Hibernate DDL generation MUST be `none` in production profile. (ADR-14)
- **BR-000-14**: `application.properties` MUST include `%dev`, `%test`, `%prod` profiles. Environment variables with `${VAR:default}` syntax MUST be used for all secrets and environment-specific values. Production secrets MUST NOT be committed. (ADR-15)
- **BR-000-15**: Production logs MUST use JSON format via `quarkus.log.console.json=true`. Every log event MUST include `traceId`, `spanId`, `correlationId` MDC fields. `System.out`, `System.err`, `printStackTrace()` MUST NOT appear in production code. (ADR-16)
- **BR-000-16**: Exception hierarchy MUST include `DomainException` and `InfrastructureException` base classes with structured error codes (`DOM-XXX`, `INF-XXX`). Throwing raw `RuntimeException` or `Exception` is PROHIBITED. All exceptions MUST carry unique error codes, operation context, and correlation ID. (ADR-17)
- **BR-000-17**: OWASP security headers MUST be configured: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 1; mode=block`, `Referrer-Policy: strict-origin-when-cross-origin`, `Content-Security-Policy`, `Strict-Transport-Security`. Rate limiting configuration placeholders MUST be present. (ADR-18)
- **BR-000-18**: A dedicated `AuditLogger` category MUST be configured with a separate file handler. Audit log entries MUST follow structured JSON schema with `id`, `timestamp`, `serviceName`, `eventName`, `outcome`, `correlationId`, `traceId`, `actor`, `resource` fields. (ADR-19)
- **BR-000-19**: The Three Pillars of Observability (Metrics, Logs, Traces) MUST share trace IDs for correlation. Observability overhead MUST NOT exceed 5% of application performance. (ADR-20)
- **BR-000-20**: OpenTelemetry distributed tracing MUST be configured with Jaeger backend via OTLP gRPC (port 4317). W3C Trace Context and baggage propagation MUST be enabled. Production MUST use probabilistic sampling (1-10%). (ADR-21)
- **BR-000-21**: Micrometer with Prometheus registry MUST expose metrics at `/q/metrics`. SmallRye Health MUST expose `/q/health`, `/q/health/live`, `/q/health/ready`. Grafana dashboards MUST be provisioned with explicit datasource UIDs. Prometheus MUST use `host.docker.internal:8080` when scraping the host-running app. (ADR-22)
- **BR-000-22**: REST API errors MUST comply with RFC 9457 (Problem Details, `application/problem+json`). A reactive `ExceptionMapper<Throwable>` MUST produce structured responses with `errorCode`, `correlationId`, `traceId`, `retryable` fields. (ADR-23)
- **BR-000-23**: DTOs MUST use Bean Validation 3.0 annotations with internationalized message keys (`{validation.field.rule}`). Response DTOs SHOULD use Java records. DTOs MUST NOT expose internal implementation details. (ADR-24)
- **BR-000-24**: All user-facing and system output messages MUST be externalized to resource bundles. Supported locales: `en`, `es`, `fr`, `de`. `MessageKeys` constants class MUST provide type-safe access. Default locale MUST be configurable via `app.i18n.default-locale` property. (ADR-25, ADR-26)
- **BR-000-25**: OpenAPI annotations MUST maintain static English content. A runtime `ApiDocumentationService` MUST provide localized documentation via `/api-info` endpoint. (ADR-27)
- **BR-000-26**: Testcontainers dependencies (`testcontainers`, `clickhouse`, `junit-jupiter`) MUST be added. A `ClickHouseTestResource` implementing `QuarkusTestResourceLifecycleManager` MUST be scaffolded. (ADR-28)
- **BR-000-27**: Code generation templates MUST be documented for future business module creation, following hexagonal architecture, reactive Mutiny patterns, and established naming conventions. (ADR-29)
- **BR-000-28**: This BRD MUST follow the standard format: Summary with user story, Business Rules with RFC 2119 levels, Acceptance Criteria in Gherkin, Cross-Cutting Concerns Checklist with ADR references. (ADR-30)
- **BR-000-29**: Production builds MUST generate five artifacts: application Docker image, Flyway migration Docker image, Grafana dashboard JSON, error inventory document, OpenAPI specification. All MUST be versioned consistently. (ADR-31)
- **BR-000-30**: `./mvnw clean verify` MUST complete within 60 seconds (cold build). Application startup MUST be under 2 seconds in JVM mode. Container RSS MUST be under 256MB without business code.

## 4. Acceptance Criteria / Scenarios

```gherkin
Scenario: Reproducible build environment
  Given a fresh clone of the repository
  When I run "./setup.sh"
  Then the script exits with code 0
  And Java version is "21.0.4"
  And Maven Wrapper version is "3.9.8"
  And Docker version is 24.0+
  And Docker Compose version is 2.24+

Scenario: Successful full build
  Given all prerequisites are satisfied
  When I run "./mvnw clean verify"
  Then the build exits with code 0
  And Spotless formatting check passes
  And Checkstyle check passes
  And PMD check passes
  And SpotBugs check passes
  And all unit and architecture tests pass

Scenario: Multi-module structure compliance
  Given the project is built
  When I inspect the module structure
  Then modules "platform-api", "platform-infrastructure", "exception-library", "i18n-library", "container-app" exist
  And no module named "shared-library", "common-library", or "utils-library" exists
  And ArchUnit tests verify hexagonal dependency direction

Scenario: Application starts and exposes operational endpoints
  Given the application is built with "./mvnw package -pl container-app -am"
  When I start the application
  Then it starts in under 2 seconds
  And "GET /q/health" returns {"status":"UP"}
  And "GET /q/metrics" returns Prometheus metrics containing "jvm_memory_used_bytes"
  And "GET /q/openapi" returns a valid OpenAPI specification
  And "GET /internal/diagnostics" returns an operational status response

Scenario: Security headers are present on all responses
  Given the application is running
  When I send "HEAD /" to the application
  Then the response includes header "X-Content-Type-Options: nosniff"
  And the response includes header "X-Frame-Options: DENY"
  And the response includes header "X-XSS-Protection: 1; mode=block"
  And the response includes header "Referrer-Policy: strict-origin-when-cross-origin"

Scenario: Structured JSON logging with correlation
  Given the application is running in prod profile
  When I send a request with header "X-Correlation-Id: test-corr-123"
  Then the application log output is valid JSON
  And the log entry contains field "correlationId" with value "test-corr-123"
  And the log entry contains fields "traceId" and "spanId"

Scenario: Distributed tracing is operational
  Given the application and Jaeger are running via Docker Compose
  When I send a request to "/q/health"
  Then a trace appears in Jaeger UI at "http://localhost:16686"
  And the trace contains service name matching "quarkus.application.name"

Scenario: Metrics collection and Grafana integration
  Given the full Docker Compose stack is running (app, Prometheus, Grafana)
  When I query Prometheus targets at "http://localhost:9090/api/v1/targets"
  Then the application target shows health "up"
  And Grafana is accessible at "http://localhost:3000"
  And provisioned dashboards are visible

Scenario: Docker Compose development stack
  Given Docker and Docker Compose are installed
  When I run "make start" (or equivalent Docker Compose up)
  Then all services start: ClickHouse, Tabix, Jaeger, Prometheus, Grafana
  And all services reach "healthy" state within 2 minutes
  And "make stop" cleanly shuts down all services

Scenario: Internationalization foundation
  Given the project is built
  When I inspect the resource bundle directories
  Then "i18n/" directory exists with message property files for en, es, fr, de
  And "MessageKeys.java" compiles with Validation, Log, Api, Entity, Domain inner classes
  And validation annotations reference message keys via "{validation.field.rule}" pattern

Scenario: ClickHouse database connectivity
  Given Docker Compose stack is running with ClickHouse
  When I connect to ClickHouse at "http://localhost:8123"
  Then ClickHouse responds to "SELECT 1" query
  And Tabix Web UI is accessible at "http://localhost:8124"

Scenario: Flyway migration scaffolding
  Given the project is built
  When I inspect the container-app resources
  Then directory "db/migration" exists
  And Flyway configuration is present in application.properties
  And production profile sets "quarkus.hibernate-orm.database.generation=none"

Scenario: Testcontainers scaffolding
  Given the project is built
  When I inspect test dependencies
  Then "testcontainers", "clickhouse" (TC module), and "junit-jupiter" (TC) are present
  And "ClickHouseTestResource" class exists in test sources

Scenario: Exception handling foundation
  Given the project is built
  When I inspect exception classes
  Then "DomainException" base class exists with errorCode and ErrorContext
  And "InfrastructureException" base class exists
  And a reactive ExceptionMapper producing "application/problem+json" is configured

Scenario: Audit logging stub
  Given the application is running
  When I inspect logging configuration
  Then "AuditLogger" category is configured with a separate file handler
  And audit log entries follow the structured JSON schema

Scenario: Configuration management
  Given the application properties file exists
  When I inspect the configuration
  Then "%dev", "%test", "%prod" profile prefixes are present
  And environment variable substitution "${VAR:default}" is used for secrets
  And no plaintext production passwords are committed

Scenario: Validation scripts pass
  Given the full Docker Compose stack is running with the application
  When I run "scripts/validate-deployment.sh"
  Then the script exits with code 0
  When I run "scripts/validate-observability.sh"
  Then the script exits with code 0
  When I run "scripts/validate-compose.sh"
  Then the script exits with code 0
  When I run "scripts/validate-security-headers.sh"
  Then the script exits with code 0
```

## 5. Non-Functional Requirements

| Category | Requirement | Target | ADR Reference |
|----------|------------|--------|---------------|
| Startup | JVM mode startup time | < 2 seconds | ADR-08 |
| Memory | Container RSS (baseline, no business code) | < 256 MB | ADR-08 |
| Build Time | `./mvnw clean verify` cold build | < 60 seconds | ADR-03 |
| Logging Overhead | Structured logging performance tax | < 5% CPU increase | ADR-16 |
| Trace Latency | Added latency per simple request | < 2 ms | ADR-21 |
| Observability Overhead | Total observability infrastructure tax | < 5% of application performance | ADR-20 |
| Validation Scripts | Total execution time | < 15 seconds | ADR-04 |
| Container Startup | Docker Compose full stack ready | < 2 minutes | ADR-07 |
| Dashboard Query | Grafana dashboard query response | < 5 seconds | ADR-22 |
| Security | No sensitive data (PII, credentials) in logs | Manual inspection + redaction utility | ADR-16, ADR-18 |
| Audit Latency | Audit log event overhead per request | < 5 ms | ADR-19 |

## 6. Implementation Phases

| Phase | Activities | Deliverables |
|-------|------------|--------------|
| P1 Project Scaffolding | Parent POM with Quarkus 3.32.2 BOM, modules, `.sdkmanrc`, Maven Wrapper, `.gitignore` | `pom.xml`, module directories, `.sdkmanrc`, `mvnw` |
| P2 Configuration & Quality | `application.properties` with profiles, Spotless, Checkstyle, PMD, SpotBugs plugins | Config files, `build/pmd-ruleset.xml` |
| P3 Architecture Foundation | Hexagonal package structure, ArchUnit tests, exception hierarchy (`DomainException`, `InfrastructureException`), error code registry | Package directories, `ArchitectureTest.java`, exception classes |
| P4 Observability Stack | Logging filter with correlation IDs, OpenTelemetry config, Micrometer Prometheus metrics, health checks | `LoggingRequestFilter`, OTEL config, metric bean |
| P5 Security Baseline | OWASP headers in config, rate limiting placeholders, JWT key generation scaffolding, input validation service stub | `application.properties` security section, RSA key stubs |
| P6 i18n Foundation | Resource bundle directories, `MessageKeys` class, `LocaleContextProvider`, `MessageResolver`, `LocalizedLogger` stubs, API doc localization service | `i18n/` bundles, Java classes |
| P7 Database & Migration | ClickHouse client config class, Flyway dependency, `db/migration` directory, Testcontainers `ClickHouseTestResource` | Config classes, migration directory, test resource |
| P8 Audit Logging | `AuditLog` domain model, `AuditLogService` with async Mutiny execution, dedicated logger configuration | Audit classes, config |
| P9 Error Handling | RFC 9457 `ProblemDetail` model, `HttpStatusMapper`, reactive `ExceptionMapper`, DTO validation samples | Error handling classes |
| P10 Containerization | Multi-stage `Dockerfile`, `.dockerignore`, `docker-compose.yml` (ClickHouse, Tabix, Jaeger, Prometheus, Grafana), Prometheus config, Grafana provisioning | Docker artifacts, compose file |
| P11 Automation | `setup.sh`, `Makefile` with auto-detection, validation scripts (`validate-deployment.sh`, `validate-observability.sh`, `validate-compose.sh`, `validate-security-headers.sh`) | Scripts, Makefile |
| P12 Documentation & Verification | `README.md`, this BRD, run all validation scripts, verify all success criteria | Docs, validation reports |

## 7. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Java Runtime | ADR-02 | JDK 21.0.4 (Eclipse Temurin) pinned via `.sdkmanrc`; `java -version` validated in `setup.sh` | TBD |
| Build Automation | ADR-03 | Maven Wrapper 3.9.8 committed; all commands use `./mvnw`; version validated in `setup.sh` | TBD |
| Toolchain Pinning | ADR-04 | `.sdkmanrc` + `.mvn/wrapper/maven-wrapper.properties` pin exact versions; `setup.sh` fast-fails on mismatch | TBD |
| Containerization | ADR-05 | Docker 24.0+ / Compose 2.24+ required; scripts support both `docker compose` and `docker-compose` variants | TBD |
| Dev Environment Automation | ADR-06 | `Makefile` with auto-detected Compose command; `setup.sh` validates prerequisites; multi-module `-am` flag in dev target | TBD |
| Containerized Dev Environment | ADR-07 | Docker Compose with ClickHouse, Tabix, Jaeger, Prometheus, Grafana; health checks; `host.docker.internal` for Prometheus scraping | TBD |
| Application Framework | ADR-08 | Quarkus 3.32.2 BOM; RESTEasy Reactive (`quarkus-rest`); Mutiny reactive patterns; `quarkus-resteasy` FORBIDDEN | TBD |
| Multi-Module Structure | ADR-09 | Parent + 5 modules; single-responsibility libraries; `shared-library` FORBIDDEN; Maven Enforcer bans god libraries; `-am` flag documented | TBD |
| Hexagonal Architecture | ADR-10 | `domain/`, `port/`, `adapter/` packages; ArchUnit enforces inward dependency direction; ports are interfaces only | TBD |
| Package Structure | ADR-11 | `com.copilot.quarkus.[layer].[context].[component]`; lowercase package names; ArchUnit validates boundaries | TBD |
| Code Quality | ADR-12 | Spotless (Google Java Format), Checkstyle (Google Checks), PMD (custom ruleset: CyclomaticComplexity ≤ 10, CognitiveComplexity ≤ 15), SpotBugs with FindSecBugs; all fail build on violation | TBD |
| Primary Database | ADR-13 | ClickHouse 24.x in Docker Compose (ports 8123, 9000); `ClickHouseClientConfig` class; Tabix at port 8124; batch insert patterns | TBD |
| Data Migration | ADR-14 | Flyway dependency added; `db/migration` directory scaffolded; `%prod.quarkus.hibernate-orm.database.generation=none` | TBD |
| Configuration Management | ADR-15 | `application.properties` with `%dev/%test/%prod` profiles; `${VAR:default}` for secrets; security headers; CORS; OpenAPI config | TBD |
| Structured Logging | ADR-16 | `%prod.quarkus.log.console.json=true`; MDC fields (`traceId`, `spanId`, `correlationId`); `LoggingRequestFilter`; `SensitiveValueSanitizer` placeholder; `System.out`/`printStackTrace` FORBIDDEN | TBD |
| Exception Handling | ADR-17 | `DomainException`, `InfrastructureException` hierarchies; `ErrorContext` with error codes; `@RegisterForReflection` for GraalVM; no raw `RuntimeException` | TBD |
| Security (OWASP) | ADR-18 | OWASP headers configured; rate limiting placeholders; JWT RS256 key scaffolding; `InputValidationService` stub; OWASP Dependency Check plugin; SSRF prevention patterns | TBD |
| Audit Logging | ADR-19 | `AuditLog` domain model; `AuditLogService` with async Mutiny worker pool; dedicated `AuditLogger` file handler; structured JSON schema; PII masking placeholder | TBD |
| Observability Strategy | ADR-20 | Three Pillars (Metrics + Logs + Traces) share trace IDs; unified Grafana visualization; < 5% overhead; custom span attributes | TBD |
| Distributed Tracing | ADR-21 | `quarkus-opentelemetry` extension; OTLP gRPC to Jaeger (port 4317); W3C Trace Context; `@WithSpan` for manual instrumentation; production sampling 1-10% | TBD |
| Monitoring & Dashboards | ADR-22 | Micrometer Prometheus at `/q/metrics`; Grafana dashboards provisioned with explicit datasource UIDs; Prometheus scrapes `host.docker.internal:8080`; alert rules defined | TBD |
| REST API Error Handling | ADR-23 | RFC 9457 `ProblemDetail` model; `ReactiveExceptionMapper`; `HttpStatusMapper`; `application/problem+json` content type; error codes mapped to HTTP statuses | TBD |
| DTO Validation | ADR-24 | Bean Validation 3.0; internationalized `{message.key}` annotations; Java records for responses; `CreateXxxRequest`/`XxxResponse` naming; `@RegisterForReflection` | TBD |
| Internationalization | ADR-25 | Locale-independent error codes (`DOMAIN-CATEGORY-SEQUENCE`); resource bundles per domain; `LocaleContextProvider` with reactive propagation; `MessageResolver` with `ConcurrentHashMap` cache | TBD |
| Message Externalization | ADR-26 | All strings externalized; `MessageKeys` constants (Validation, Log, Api, Entity, Domain); `LocalizedLogger` wrapping SLF4J; direct string logging PROHIBITED; configurable default locale via `app.i18n.default-locale` | TBD |
| API Doc Localization | ADR-27 | Static OpenAPI annotations with English; runtime `ApiDocumentationService`; `/api-info` endpoint with localized output; message keys for all API docs | TBD |
| Testcontainers | ADR-28 | `testcontainers`, `clickhouse` TC module, `junit-jupiter` TC in test scope; `ClickHouseTestResource` implementing `QuarkusTestResourceLifecycleManager`; Ryuk configurable | TBD |
| Code Generation Templates | ADR-29 | Templates documented for domain entity, repository port, application service, REST controller, DTO patterns; referenced for future use | TBD |
| BRD Format | ADR-30 | This BRD follows standard format: Summary with user story, Business Rules with RFC 2119, Gherkin scenarios, Cross-Cutting Concerns Checklist | TBD |
| Production Build Artifacts | ADR-31 | Build pipeline scaffolded for 5 artifacts: app image, migration image, Grafana JSON, error inventory, OpenAPI spec; multi-stage Dockerfiles; consistent versioning | TBD |

## 8. Validation & Verification Scripts

| Script | Purpose | Success Criteria | ADR Reference |
|--------|---------|------------------|---------------|
| `scripts/setup.sh` | Tool & wrapper version enforcement | Exits 0; Java 21.0.4, Maven 3.9.8, Docker 24.0+, Compose 2.24+ | ADR-04 |
| `scripts/validate-deployment.sh` | App health, metrics, headers, OpenAPI | All endpoints reachable & checks pass | ADR-08, ADR-22 |
| `scripts/validate-observability.sh` | Emit test request and confirm trace & log fields | Trace appears in Jaeger; log contains correlationId, traceId | ADR-20, ADR-21 |
| `scripts/validate-compose.sh` | Ensure all compose services healthy | All services in `healthy` state | ADR-05, ADR-07 |
| `scripts/validate-security-headers.sh` | Confirm required OWASP security headers | `curl -I` includes all required headers | ADR-18 |

## 9. Success Criteria Consolidated from ADRs

| ADR | Criteria | Validation Method | Expected Result |
|-----|----------|-------------------|-----------------|
| ADR-02 | JDK Version | `java -version 2>&1 \| head -1` | Contains `21.0.4` |
| ADR-02 | SDKMAN Config | `cat .sdkmanrc` | `java=21.0.4-tem` |
| ADR-03 | Maven Wrapper | `./mvnw -v` | Maven 3.9.8 |
| ADR-03 | Clean Build | `./mvnw clean package -DskipTests` | Exit code 0 |
| ADR-04 | .sdkmanrc Present | `test -f .sdkmanrc` | File exists |
| ADR-04 | Setup Script | `./setup.sh` | Exit code 0 |
| ADR-05 | Docker Installed | `docker --version` | Version 24.0+ |
| ADR-05 | Docker Compose | `docker compose version` or `docker-compose version` | Version 2.24+ |
| ADR-05 | Services Start | `docker compose up -d` | All services healthy |
| ADR-07 | docker-compose.yml | `ls local-env/docker-compose*.yml` | File exists |
| ADR-07 | Health Checks | `grep healthcheck docker-compose.yml` | Health checks configured |
| ADR-07 | Host-to-Container | Prometheus scrape | Metrics returned from `host.docker.internal:8080` |
| ADR-08 | Quarkus Version | `grep quarkus.platform.version pom.xml` | 3.32.2 |
| ADR-08 | BOM Import | `grep quarkus-bom pom.xml` | Present in dependencyManagement |
| ADR-08 | Health Check | `curl localhost:8080/q/health` | `{"status":"UP"}` |
| ADR-08 | Metrics | `curl localhost:8080/q/metrics` | Prometheus metrics |
| ADR-08 | OpenAPI | `curl localhost:8080/q/openapi` | Valid OpenAPI spec |
| ADR-08 | Startup Time | Application logs | < 2 seconds (JVM) |
| ADR-09 | Module Structure | `find . -name "pom.xml"` | Business/API/Library modules exist |
| ADR-09 | No God Libraries | `grep -l "shared-library"` | No matches |
| ADR-09 | ArchUnit Tests | `./mvnw test -Dtest=*ArchTest*` | All pass |
| ADR-10 | ArchUnit Dependency | `grep archunit pom.xml` | Present |
| ADR-10 | Domain Independence | ArchUnit test | Domain has no adapter imports |
| ADR-10 | Hexagonal Tests | `./mvnw test -Dtest=*Hexagonal*` | All pass |
| ADR-11 | Domain Package | `find . -path "*domain/model*"` | Exists |
| ADR-11 | Infrastructure Package | `find . -path "*infrastructure/adapter*"` | Exists |
| ADR-12 | Spotless | `./mvnw spotless:check` | Exit code 0 |
| ADR-12 | Checkstyle | `./mvnw checkstyle:check` | Exit code 0 |
| ADR-12 | PMD | `./mvnw pmd:check` | Exit code 0 |
| ADR-12 | SpotBugs | `./mvnw spotbugs:check` | Exit code 0 |
| ADR-13 | ClickHouse Running | `clickhouse-client --query "SELECT 1"` | Returns 1 |
| ADR-14 | Flyway Dependency | `grep flyway pom.xml` | Present |
| ADR-14 | Migration Directory | `ls src/main/resources/db/migration` | Directory exists |
| ADR-15 | Profiles | `grep '%dev\|%test\|%prod' application.properties` | Present |
| ADR-15 | Env Var Substitution | `grep '\${' application.properties` | Used for secrets |
| ADR-16 | JSON Logging Config | `grep quarkus.log.console.json application.properties` | Enabled for prod |
| ADR-16 | Correlation Fields | Log output | `correlationId`, `traceId`, `spanId` present |
| ADR-16 | No System.out | `grep -r "System.out" src/main/java` | Zero matches |
| ADR-17 | Exception Hierarchy | Code inspection | `DomainException`, `InfrastructureException` exist |
| ADR-18 | Security Headers | `curl -I localhost:8080` | All OWASP headers present |
| ADR-19 | AuditLog Entity | `grep -r "class AuditLog" src/main/java` | Exists |
| ADR-19 | Dedicated Logger | `grep AuditLogger application.properties` | Separate handler configured |
| ADR-20 | OTel Config | `grep quarkus.otel application.properties` | Configured |
| ADR-20 | Prometheus Scrape | Prometheus targets | App target UP |
| ADR-21 | Traces in Jaeger | Jaeger UI | Traces visible |
| ADR-22 | Grafana Accessible | `curl localhost:3000` | UI loads |
| ADR-22 | Dashboard Provisioned | Grafana dashboards API | Dashboards exist |
| ADR-23 | ProblemDetail | Code inspection | RFC 9457 model exists |
| ADR-24 | Validation Annotations | Code inspection | `@NotBlank`, `@Size` with message keys |
| ADR-25 | Resource Bundles | `ls i18n/` | Bundle files for en, es, fr, de |
| ADR-26 | MessageKeys Class | `grep "class MessageKeys"` | Compiled with inner classes |
| ADR-27 | API Info Endpoint | `curl localhost:8080/api-info` | Localized response |
| ADR-28 | Testcontainers Dep | `grep testcontainers pom.xml` | Present |
| ADR-28 | ClickHouse TC | `grep ClickHouseTestResource src/test/java` | Class exists |
| ADR-30 | BRD Format | BRD file inspection | Summary, Rules, Gherkin, Checklist present |
| ADR-31 | Dockerfile | `ls Dockerfile` | Multi-stage build, non-root user |
| Full Build | All Criteria | `./mvnw clean verify` | Exit code 0 |

## 10. Requirements Coverage Matrix

| Requirement | Implementing Artifact | Governing ADR(s) | Status |
|------------|----------------------|-------------------|--------|
| Java 21 reproducible runtime | `.sdkmanrc`, `setup.sh` | ADR-02, ADR-04 | TBD |
| Maven build automation | Maven Wrapper 3.9.8, `pom.xml` | ADR-03 | TBD |
| Toolchain version enforcement | `setup.sh`, `.sdkmanrc`, `.mvn/wrapper/` | ADR-04 | TBD |
| Docker containerization | `Dockerfile`, `.dockerignore` | ADR-05, ADR-31 | TBD |
| Development automation | `Makefile`, `setup.sh` | ADR-06 | TBD |
| Local dev stack | `docker-compose.yml` (ClickHouse, Tabix, Jaeger, Prometheus, Grafana) | ADR-07 | TBD |
| Quarkus 3.32.2 framework | Parent POM, BOM import, extensions | ADR-08 | TBD |
| Multi-module architecture | Parent + 5 modules, no god libraries | ADR-09 | TBD |
| Hexagonal architecture | Package structure, ArchUnit tests | ADR-10 | TBD |
| Package conventions | `domain/application/infrastructure` layers | ADR-11 | TBD |
| Code quality toolchain | Spotless, Checkstyle, PMD, SpotBugs | ADR-12 | TBD |
| ClickHouse database | Docker Compose service, client config | ADR-13 | TBD |
| Flyway migrations | Dependency, `db/migration` directory | ADR-14 | TBD |
| Configuration management | `application.properties` with profiles | ADR-15 | TBD |
| Structured JSON logging | JSON config, correlation filter, MDC | ADR-16 | TBD |
| Exception hierarchy | `DomainException`, `InfrastructureException`, error codes | ADR-17 | TBD |
| OWASP security headers | `application.properties` header config | ADR-18 | TBD |
| Audit logging | `AuditLog` model, `AuditLogService`, dedicated handler | ADR-19 | TBD |
| Observability strategy | Three pillars integration, correlation | ADR-20 | TBD |
| Distributed tracing | OpenTelemetry + Jaeger, OTLP config | ADR-21 | TBD |
| Monitoring dashboards | Prometheus metrics, Grafana provisioning | ADR-22 | TBD |
| RFC 9457 error handling | `ProblemDetail`, `ExceptionMapper` | ADR-23 | TBD |
| DTO validation | Bean Validation 3.0, i18n message keys | ADR-24 | TBD |
| Internationalization | Resource bundles, error codes, `LocaleContextProvider` | ADR-25 | TBD |
| Message externalization | `MessageKeys`, `LocalizedLogger`, bundle files | ADR-26 | TBD |
| API doc localization | `ApiDocumentationService`, `/api-info` | ADR-27 | TBD |
| Testcontainers | Dependencies, `ClickHouseTestResource` | ADR-28 | TBD |
| Code generation templates | Documented templates for future use | ADR-29 | TBD |
| BRD format compliance | This document | ADR-30 | TBD |
| Production build pipeline | 5 artifact scaffolding | ADR-31 | TBD |
| Validation scripts | `scripts/validate-*.sh` | ADR-04, ADR-06 | TBD |
| README documentation | `README.md` | ADR-06 | TBD |

## 11. Related ADRs

- ADR-00: ADR Structure Summary — Template rules and RFC 2119 standards
- ADR-02: Java Runtime Environment — JDK 21.0.4, SDKMAN, virtual threads
- ADR-03: Build Automation — Maven Wrapper 3.9.8, dependency management
- ADR-04: Toolchain Version Management — Exact version pinning, `setup.sh`
- ADR-05: Containerization Infrastructure — Docker 24.0+, Compose 2.24+, dual command support
- ADR-06: Development Environment Automation — Makefile, setup scripts, `-am` flag
- ADR-07: Containerized Development Environment — Graduated strategy, `host.docker.internal`
- ADR-08: Application Framework Quarkus — Quarkus 3.32.2, RESTEasy Reactive, Mutiny
- ADR-09: Multi-Module Structure — Module types, single-responsibility libraries, forbidden names
- ADR-10: Software Architecture Hexagonal — Ports and Adapters, ArchUnit enforcement
- ADR-11: Package Structure Organization — `domain/application/infrastructure` convention
- ADR-12: Code Quality Standards — Spotless, Checkstyle, PMD, SpotBugs, FindSecBugs
- ADR-13: Primary Database ClickHouse — ClickHouse 24.x, columnar storage, Tabix
- ADR-14: Data Migration Strategy — Flyway, versioned SQL scripts, `db/migration`
- ADR-15: Application Configuration Management — Profiles, env vars, security defaults
- ADR-16: Structured Logging Strategy — JSON logging, MDC correlation, redaction
- ADR-17: Exception Handling Reactive — Typed hierarchy, error codes, Mutiny integration
- ADR-18: Security Implementation OWASP — Headers, JWT scaffolding, rate limiting, SSRF prevention
- ADR-19: Audit Logging Strategy — Async audit, dedicated handler, JSON schema
- ADR-20: Observability Strategy — Three Pillars, trace ID correlation
- ADR-21: Distributed Tracing — OpenTelemetry, Jaeger, OTLP, sampling
- ADR-22: Application Monitoring Observability — Prometheus, Grafana, datasource UIDs, alerts
- ADR-23: REST API Error Handling — RFC 9457, ProblemDetail, reactive mapper
- ADR-24: Data Transfer Validation — Bean Validation 3.0, i18n messages, Java records
- ADR-25: Internationalization Strategy — Error codes, resource bundles, reactive context
- ADR-26: Message Externalization — MessageKeys, LocalizedLogger, configurable locale
- ADR-27: API Documentation Localization — Static OpenAPI + runtime localization
- ADR-28: Testcontainers Integration — ClickHouse TC, QuarkusTestResource
- ADR-29: Code Generation Templates — Entity, port, service, controller, DTO templates
- ADR-30: Business Requirement Format — BRD structure, Gherkin, traceability
- ADR-31: Production Build Artifacts — 5 mandatory artifacts, multi-stage Docker

## 12. Open Questions / Future Enhancements

| Item | Decision Needed By | Notes |
|------|--------------------|-------|
| GraalVM native image enablement | Post first business module | Requires native-image container stage and reflection config |
| Audit log sink choice | Before first business event (BR-001) | Evaluate ClickHouse table vs separate file vs external collector |
| Rate limiting implementation | Prior to external API exposure | Consider bucket4j or Quarkus interceptor per ADR-18 |
| Service mesh headers | When deploying to Kubernetes | Align tracing headers with mesh (e.g., Istio) per ADR-21 |
| JWT authentication flows | Before first secured business endpoint | Full OIDC/JWT flow per ADR-18 |
| Log aggregation backend | Before production deployment | Evaluate Loki, ELK, or Fluent Bit per ADR-16 |

## 13. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Over-engineering before business features | Delayed delivery | Keep modules minimal; no business code; stubs only |
| Tool version drift across team | Build failures | `setup.sh` fast-fails + `.sdkmanrc` + Maven Wrapper |
| Observability misconfiguration | Debugging complexity | Validation scripts exercising all endpoints |
| Architecture erosion in future modules | Hard refactors | ArchUnit baseline in place early; tests fail build |
| ClickHouse dev/prod parity | Query behavior differences | Testcontainers use same Docker image as Compose |
| Docker Compose variant incompatibility | Script failures on different platforms | Auto-detection pattern for both `docker compose` and `docker-compose` |

## 14. Approval

| Role | Name | Date | Status |
|------|------|------|--------|
| Tech Lead | TBD | TBD | Pending |
| Architect | TBD | TBD | Pending |
| Product Owner | TBD | TBD | Pending |

---
End of BR-000.
