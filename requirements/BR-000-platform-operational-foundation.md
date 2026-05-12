# BR-000: Platform Operational Foundation

## 1. Summary

As a **developer**, I want **a fully configured development environment with all required tools and infrastructure services installed and validated**, so that **I can immediately begin building business features on a reproducible, observable platform without manual environment troubleshooting**.

## 2. Business Rules

- **BR-000-01**: Implement ADR-02 (Java Runtime Environment) — JDK 21.0.4 Eclipse Temurin via SDKMAN
- **BR-000-02**: Implement ADR-03 (Build Automation) — Maven 3.9.8 with Maven Wrapper
- **BR-000-03**: Implement ADR-04 (Toolchain Version Management) — .sdkmanrc with pinned versions
- **BR-000-04**: Implement ADR-05 (Containerization Infrastructure) — Docker 24.0+, Docker Compose 2.24+
- **BR-000-05**: Implement ADR-06 (Development Environment Automation) — Makefile, setup.sh, Docker Compose services
- **BR-000-06**: Implement ADR-07 (Containerized Development Environment) — Graduated approach with host.docker.internal
- **BR-000-07**: Implement ADR-08 (Application Framework) — Quarkus 3.32.2 with BOM
- **BR-000-08**: Implement ADR-13a (ClickHouse Infrastructure) — ClickHouse 24.3 with Tabix Web UI

## 3. Acceptance Criteria / Scenarios

```gherkin
Scenario: Validate Java runtime environment
  Given JDK 21.0.4 Eclipse Temurin is installed via SDKMAN
  When I run "java -version"
  Then the output contains "21.0.4"

Scenario: Validate Maven Wrapper
  Given Maven Wrapper files exist (mvnw, maven-wrapper.jar, maven-wrapper.properties)
  When I run "./mvnw -v"
  Then the output contains "Apache Maven 3.9.8"

Scenario: Validate Docker infrastructure
  Given Docker and Docker Compose are installed
  When I run "make start"
  Then ClickHouse, Jaeger, Prometheus, and Grafana services are running

Scenario: Validate infrastructure health
  Given Infrastructure services are started
  When I check service endpoints
  Then ClickHouse responds to ping, Jaeger UI is accessible, Prometheus is healthy, Grafana is accessible

Scenario: Validate build succeeds
  Given all prerequisite tools are installed
  When I run "make build"
  Then the build completes with exit code 0

Scenario: Validate validation script passes
  Given BR-000 implementation is complete
  When I run "bash scripts/validate-br000.sh"
  Then all 53 checks pass with zero failures
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Java Runtime | ADR-02, ADR-04 | JDK 21.0.4-tem via .sdkmanrc | Yes |
| Build Tooling | ADR-03, ADR-12 | Maven 3.9.8, Spotless 3.4.0, SpotBugs 4.9.8.3 | Yes |
| Containerization | ADR-05, ADR-06 | Docker 24.0+, Compose auto-detection | Yes |
| Database | ADR-13a, ADR-13b | ClickHouse 24.3-alpine with JDBC 0.6.3 | Yes |
| Observability | ADR-20, ADR-21, ADR-22 | Jaeger, Prometheus, Grafana with explicit UIDs | Yes |
| Security | ADR-18 | OWASP headers, JWT RS256 | N/A (framework level) |
| Configuration | ADR-15 | Profile-based application.properties | N/A (framework level) |
| Logging | ADR-16 | JSON structured logging | N/A (framework level) |
| Exception Handling | ADR-17 | Layered exception hierarchy | N/A (framework level) |
| Package Structure | ADR-11 | Hexagonal architecture packages | N/A (framework level) |
| Multi-Module | ADR-09 | Parent POM with dependency management | Yes |

## 5. Related ADRs

- **ADR-02**: Java Runtime Environment — JDK 21.0.4 Eclipse Temurin
- **ADR-03**: Build Automation — Maven 3.9.8 with Maven Wrapper
- **ADR-04**: Toolchain Version Management — SDKMAN .sdkmanrc pinning
- **ADR-05**: Containerization Infrastructure — Docker 24.0+, Compose 2.24+
- **ADR-06**: Development Environment Automation — Makefile, setup.sh
- **ADR-07**: Containerized Development Environment — Graduated approach
- **ADR-08**: Application Framework — Quarkus 3.32.2
- **ADR-09**: Multi-Module Maven Architecture — Parent POM structure
- **ADR-10**: Software Architecture (Hexagonal) — Ports & Adapters
- **ADR-11**: Package Structure Organization — Hexagonal package layout
- **ADR-12**: Code Quality Standards — Spotless, Checkstyle, PMD, SpotBugs
- **ADR-13a**: ClickHouse Infrastructure Setup — Docker Compose service
- **ADR-13b**: ClickHouse Coding Standards — Schema design, reactive adapters
- **ADR-14**: Data Migration Strategy — Flyway migrations
- **ADR-15**: Application Configuration Management — Profile-based properties
- **ADR-16**: Structured Logging Strategy — JSON logging with MDC
- **ADR-17**: Exception Handling (Reactive) — Layered exception hierarchy
- **ADR-18**: Security Implementation (OWASP) — JWT, RBAC, headers
- **ADR-19**: Audit Logging Strategy — Async structured audit logs
- **ADR-20**: Observability Strategy — Three pillars (Metrics, Logs, Traces)
- **ADR-21**: Distributed Tracing — OpenTelemetry with Jaeger
- **ADR-22**: Application Monitoring & Observability — Grafana dashboards
- **ADR-23**: REST API Error Handling — RFC 9457 Problem Details
- **ADR-24**: Data Transfer & Validation — DTO patterns, Bean Validation
- **ADR-25**: Internationalization Strategy — Resource bundles, locale context
- **ADR-26**: Message Externalization — Complete string externalization
- **ADR-27**: API Documentation Localization — OpenAPI localization
- **ADR-28**: Testcontainers Integration — ClickHouse test containers
- **ADR-29**: Code Generation Templates — Implementation consistency
- **ADR-30**: Business Requirement Format — BRD standard format
- **ADR-31**: Production Build Artifacts — Five mandatory artifacts

## 6. Installation Steps

1. **Prerequisites**: Install JDK 21.0.4 (Eclipse Temurin), Docker 24.0+, Docker Compose 2.24+
2. **SDKMAN**: Install SDKMAN and run `sdk env install` to get pinned JDK/Maven versions
3. **Maven Wrapper**: Verify `./mvnw -v` returns Maven 3.9.8
4. **Infrastructure**: Run `make start` to start ClickHouse, Tabix, Jaeger, Prometheus, Grafana
5. **Validation**: Run `bash scripts/validate-br000.sh` to verify all 53 checks pass
6. **Development**: Run `make dev` to start Quarkus development server

## 7. Verification

- Each business rule relates to an ADR. Success Criteria in each ADR **MUST** be satisfied.
- All shell scripts **MUST** be verified to be working correctly (syntax check + runtime validation).
- `make build` **MUST** complete with exit code 0.
- `bash scripts/validate-br000.sh` **MUST** report 53 passed, 0 failed.
- Infrastructure services (ClickHouse, Jaeger, Prometheus, Grafana) **MUST** be healthy and accessible.
