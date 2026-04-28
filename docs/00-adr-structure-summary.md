# ADR Structure: Optimized for Architectural Decisions Records

## ADR Template Rules:
This document and all ADRs derived from this ADR **MUST** follow RFC 2119 standards for requirement levels:

- **MUST** / **REQUIRED** / **SHALL**: Absolute requirement
- **MUST NOT** / **SHALL NOT**: Absolute prohibition  
- **SHOULD** / **RECOMMENDED**: Strong recommendation with rare exceptions
- **SHOULD NOT** / **NOT RECOMMENDED**: Strong recommendation against with rare exceptions
- **MAY** / **OPTIONAL**: Truly optional, implementation choice
- Every ADR uses the following structure, designed for both human and LLM agent automation. 
- Every text in ADR uses the following structure **MUST** be short and clear.
- Every ADR derived from this template **MUST** be objective, evidence-based and quality metrics and contents **SHOULD NOT** be open to interpretation

## ADR Template

1. **# Issue:** Short issue title
	- Problem statement and context (for developer use to understand requirements)
	- Clearly define problem scope and impact
	- **MUST** be short
2. **# Decision:**
	- Specific solution and implementation details (**MUST** follow exactly)
	- **MUST** Provide concrete, actionable decisions
3. **# Ownership:**
	- **MUST** define a table with columns: Role, Person, Competencies
	- **MUST** include a **Responsible** role identifying who owns creating, editing, and maintaining the ADR
	- **MUST** include an **Approver** role identifying who validates architectural trade-offs and approves changes
	- Competencies column **MUST** list specific technical skills relevant to the ADR scope
4. **# Constraints:**
	- Technical/business boundaries (**MUST NOT** violate)
	- **MUST NOT**  be exceeded without explicit architectural review
5. **# Alternatives:**
	- Rejected options with brief reasoning (**MUST** avoid these)
	- **MUST** Document why alternatives were not chosen
	- **MUST** have minimum 1 maximum 2 options.
	- Rejection reasoning **MUST** cite verifiable facts (official documentation, measurable metrics, documented limitations)
	- **MUST NOT** use subjective comparatives (e.g., "superior", "better", "more mature", "best") without measurable evidence
6. **# Rationale:**
	- **MUST** have justification for the decision (use for trade-off logic)
	- **MUST** have provide clear reasoning for the chosen approach
	- Justifications **MUST** reference concrete, verifiable evidence (official docs, metrics, documented constraints)
	- **MUST NOT** contain subjective adjectives or unsubstantiated comparative claims
7. **# Implementation Guidelines:**
	- **MUST** follow Step-by-step instructions (generate code/config as described)
	- **MUST** Follow precisely during implementation
	- **RECOMMENDED** Include practical examples, code snippets, configuration files
	- **RECOMMENDED** Provide exact version numbers and dependency specifications
	- **RECOMMENDED** Include setup scripts, commands, or automation tools
	- May contain troubleshooting guides and validation steps
	- Define naming conventions for all relevant artifacts (classes, methods, files, packages, etc.)
	- Specify coding standards and style requirements where applicable
8. **# Additional Recommendations:**
	- **MUST** Reference relevant documentation, tools, or resources
	- Best practices and related suggestions (apply if relevant)
	- May be applied based on specific implementation context
	- Include optimization techniques and advanced configurations
	
9. **# Success Criteria:**
	- Every compilation and coding process **MUST** successfully pass these criteria.
	- Criteria that **MUST** all be met before considering the ADR successfully implemented.
	- **MUST** Include validation methods and expected results in table format
	- **MUST** Provide validation scripts or commands where applicable

---

## ADR Catalog

All ADRs are organized into four categories by folder. Each ADR follows the 9-section template defined above.

### 01 — Software Environment Setup

Foundation tooling, runtime, containerization, and framework decisions.

| ADR | Title | Scope |
|:----|:------|:------|
| [02](01-software-environment-setup/02-java-runtime-environment.md) | Java Runtime Environment | JDK 21 (Eclipse Temurin 21.0.4), SDKMAN management |
| [03](01-software-environment-setup/03-build-automation.md) | Build Automation | Maven 3.9.8, Maven Wrapper, dependency version properties |
| [04](01-software-environment-setup/04-toolchain-version-management.md) | Toolchain Version Management | Exact version pinning, `setup.sh` validation script |
| [05](01-software-environment-setup/05-containerization-infrastructure.md) | Containerization Infrastructure | Docker 24.0+, Docker Compose 2.24+ |
| [06](01-software-environment-setup/06-development-environment-automation.md) | Development Environment Automation | Makefile, `setup.sh`, Docker Compose services (ClickHouse, Jaeger, Prometheus, Grafana) |
| [07](01-software-environment-setup/07-containerized-development-environment.md) | Containerized Development Environment | Graduated approach: Dev Services → Compose → Testcontainers, `host.docker.internal` |
| [08](01-software-environment-setup/08-application-framework-quarkus.md) | Application Framework (Quarkus) | Quarkus 3.32.2, RESTEasy Reactive, Mutiny, BOM import |
| [13a](01-software-environment-setup/13a-clickhouse-infrastructure-setup.md) | ClickHouse Infrastructure Setup | ClickHouse Docker Compose, JDBC 0.6.3, Tabix Web UI, application config |

### 02 — Foundation Setup

Architecture, module structure, cross-cutting concerns, and platform-level patterns.

| ADR | Title | Scope |
|:----|:------|:------|
| [09](02-foundation-setup/09-multi-module-structure.md) | Multi-Module Maven Architecture | Business/External/API/Library/Container module types, forbidden god libraries, platform library catalog |
| [10](02-foundation-setup/10-software-architecture-hexagonal.md) | Software Architecture (Hexagonal) | Ports & Adapters, ArchUnit enforcement, domain→port→adapter dependency direction |
| [11](02-foundation-setup/11-package-structure-organization.md) | Package Structure Organization | Standardized package layout (domain/application/infrastructure), Jandex indexing, naming conventions |
| [15](02-foundation-setup/15-application-configuration-management.md) | Application Configuration Management | Comprehensive `application.properties`, profile-based config (%dev/%test/%prod), env vars |
| [16](02-foundation-setup/16-structured-logging-strategy.md) | Structured Logging Strategy | JSON logging, MDC correlation (traceId, spanId, correlationId), sensitive data redaction |
| [17](02-foundation-setup/17-exception-handling-reactive.md) | Exception Handling (Reactive) | Layered exception hierarchy (DomainException/InfrastructureException), ErrorContext, Mutiny error patterns, circuit breakers |
| [18](02-foundation-setup/18-security-implementation-owasp.md) | Security Implementation (OWASP) | JWT RS256, RBAC, input validation/sanitization, AES-256-GCM encryption, security audit logging |
| [19](02-foundation-setup/19-audit-logging-strategy.md) | Audit Logging Strategy | Structured async audit logging, Javers diffs, dedicated audit logger, AuditLog aggregate |
| [20](02-foundation-setup/20-observability-strategy.md) | Observability Strategy | Three pillars (Metrics/Logs/Traces), unified correlation, Docker Compose observability stack |
| [21](02-foundation-setup/21-distributed-tracing.md) | Distributed Tracing | OpenTelemetry, Jaeger, @WithSpan, manual span creation, sampling strategies |
| [22](02-foundation-setup/22-application-monitoring-observability.md) | Application Monitoring & Observability | Grafana dashboards, Prometheus scraping, custom metrics, alerting, datasource UID configuration |
| [23](02-foundation-setup/23-rest-api-error-handling.md) | REST API Error Handling | RFC 9457 Problem Details, ProblemDetail model, reactive exception mapper, HttpStatusMapper |
| [25](02-foundation-setup/25-internationalization-strategy.md) | Internationalization Strategy | Error codes (DOMAIN-CATEGORY-SEQUENCE), resource bundles, reactive locale context, MessageResolver |
| [26](02-foundation-setup/26-message-externalization.md) | Message Externalization | Complete string externalization, LocalizedLogger, MessageKeys constants, configurable default locale |

### 03 — Software Development Rules

Coding standards, testing, data handling, API documentation, and generation templates.

| ADR | Title | Scope |
|:----|:------|:------|
| [12](03-software-development-rules/12-code-quality-standards.md) | Code Quality Standards | Google Java Style, Spotless, Checkstyle, PMD, SpotBugs/FindSecBugs |
| [13b](03-software-development-rules/13b-clickhouse-coding-standards.md) | ClickHouse Coding Standards | Schema design, MergeTree engines, batch inserts, reactive Mutiny adapters, parameterized queries |
| [14](03-software-development-rules/14-data-migration-strategy.md) | Data Migration Strategy | Flyway migrations, `V{version}__{description}.sql` naming, quarkus-flyway extension |
| [24](03-software-development-rules/24-data-transfer-validation.md) | Data Transfer & Validation | DTO patterns, Bean Validation 3.0, Java records for responses, MessageKeys, custom validators |
| [27](03-software-development-rules/27-api-documentation-localization.md) | API Documentation Localization | OpenAPI localization, internationalized API docs for non-English markets |
| [28](03-software-development-rules/28-testcontainers-integration.md) | Testcontainers Integration | Integration testing with ClickHouse Testcontainers, `QuarkusTestResourceLifecycleManager` |
| [29](03-software-development-rules/29-code-generation-templates.md) | Code Generation Templates | Implementation consistency templates for code generation |
| [30](03-software-development-rules/30-business-requirement-format.md) | Business Requirement Format | BRD standard (`requirements/` directory, `BR-NNN-descriptive-name.md`), Gherkin scenarios, RFC 2119 rules, cross-cutting concerns checklist |

### 04 — Build Setup

Production artifact generation and deployment packaging.

| ADR | Title | Scope |
|:----|:------|:------|
| [31](04-build-setup/31-production-build-artifacts.md) | Production Build Artifacts | Coordinated build outputs: application binary, DB migrations, observability dashboards, API docs, error inventory |


## BRD Directory

Business Requirement Documents are stored in `requirements/` per ADR-30.

| BRD | Title |
|:----|:------|
| [BR-000](../requirements/BR-000-platform-operational-foundation.md) | Platform Operational Foundation |

---

## Folder Organization

```
00-adr-structure-summary.md          ← This file (template + catalog)
01-software-environment-setup/       ← Runtime, tooling, containers, framework
02-foundation-setup/                 ← Architecture, modules, cross-cutting concerns
03-software-development-rules/       ← Coding standards, testing, data, APIs
04-build-setup/                      ← Production build artifacts
requirements/                        ← Business Requirement Documents
```
