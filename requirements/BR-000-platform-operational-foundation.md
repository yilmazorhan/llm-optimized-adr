# BR-000: Platform Operational Foundation

## 1. Summary

As a **developer**, I want **a fully configured development environment with all required tools and infrastructure services installed and validated**, so that **I can immediately begin building business features on a reproducible, observable platform without manual environment troubleshooting**.

## 2. Business Rules

- **BR-000-01**: The development environment **MUST** provide JDK 21.0.4 (Eclipse Temurin) managed via SDKMAN, per ADR-02 §Decision.
- **BR-000-02**: All builds **MUST** use Maven 3.9.8 through the committed Maven Wrapper, per ADR-03 §Decision.
- **BR-000-03**: Toolchain versions **MUST** be pinned in `.sdkmanrc` and validated by `setup.sh`, per ADR-04 §Decision.
- **BR-000-04**: Containerization **MUST** use Docker 24.0+ with Docker Compose 2.24+ (Compose auto-detection), per ADR-05 §Decision.
- **BR-000-05**: Environment automation **MUST** be provided via Makefile, `setup.sh`, and Docker Compose services (Jaeger, Prometheus, Grafana), per ADR-06 §Decision.
- **BR-000-06**: The containerized development environment **MUST** follow the graduated approach (Dev Services → Compose → Testcontainers) with `host.docker.internal`, per ADR-07 §Decision.
- **BR-000-07**: All applications **MUST** use Quarkus 3.32.2 with the platform BOM import, per ADR-08 §Decision.

## 3. Acceptance Criteria / Scenarios

```gherkin
Scenario: Environment setup completes successfully
  Given a workstation with macOS or Linux and network access
  When I run "make setup" (which executes setup.sh)
  Then all pinned toolchain components are installed and validated
  And the script exits with code 0

Scenario: Active toolchain matches pinned versions
  Given .sdkmanrc pins JDK 21.0.4-tem and Maven 3.9.8
  When the setup.sh validation step runs
  Then the active JDK and Maven versions match the pinned values exactly

Scenario: Observability services are reachable
  Given Docker 24.0+ and Docker Compose 2.24+ are installed
  When I run "make start"
  Then Jaeger responds on localhost:16686
  And Prometheus responds on localhost:9090
  And Grafana responds on localhost:3000

Scenario: Quarkus dev mode boots on the platform
  When I run "make dev" in the container module
  Then Quarkus 3.32.2 starts with live reload enabled
  And the health endpoint responds at localhost:8080/q/health
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Java Runtime | ADR-02, ADR-04 | JDK 21.0.4-tem via .sdkmanrc | Yes |
| Build Tooling | ADR-03, ADR-12 | Maven 3.9.8, Spotless 3.4.0, SpotBugs 4.9.8.3 | Yes |
| Containerization | ADR-05, ADR-06 | Docker 24.0+, Compose auto-detection | Yes |
| Observability | ADR-20, ADR-21, ADR-22 | Jaeger, Prometheus, Grafana with explicit UIDs | Yes |
| Security | ADR-18 | OWASP headers, JWT RS256 | TBD (framework level) |
| Configuration | ADR-15 | Profile-based application.properties | TBD (framework level) |
| Logging | ADR-16 | JSON structured logging | TBD (framework level) |
| Exception Handling | ADR-17 | Layered exception hierarchy | TBD (framework level) |
| Package Structure | ADR-11 | Hexagonal architecture packages | TBD (framework level) |
| Multi-Module | ADR-09 | Parent POM with dependency management | Yes |
| Testing | ADR-32, ADR-33 | Surefire/Failsafe phase split and JaCoCo coverage gate wired in parent POM | TBD |

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
- **ADR-29**: Code Generation Templates — Implementation consistency
- **ADR-30**: Business Requirement Format — BRD standard format
- **ADR-31**: Production Build Artifacts — Five mandatory artifacts
- **ADR-32**: Unit Testing Strategy — JUnit 5, Mockito, AssertJ, JaCoCo coverage gate
- **ADR-33**: Integration Testing Strategy — Two-tier HSQLDB/Oracle test infrastructure
- **ADR-34**: Code Comment Standards — Traceability and algorithm explanation comments