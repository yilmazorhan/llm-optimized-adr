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
3. **# Constraints:**
	- Technical/business boundaries (**MUST NOT** violate)
	- **MUST NOT**  be exceeded without explicit architectural review
4. **# Alternatives:**
	- Rejected options with brief reasoning (**MUST** avoid these)
	- **MUST** Document why alternatives were not chosen
	- **MUST** have minimum 1 maximum 2 options.
	- Rejection reasoning **MUST** cite verifiable facts (official documentation, measurable metrics, documented limitations)
	- **MUST NOT** use subjective comparatives (e.g., "superior", "better", "more mature", "best") without measurable evidence
5. **# Rationale:**
	- **MUST** have justification for the decision (use for trade-off logic)
	- **MUST** have provide clear reasoning for the chosen approach
	- Justifications **MUST** reference concrete, verifiable evidence (official docs, metrics, documented constraints)
	- **MUST NOT** contain subjective adjectives or unsubstantiated comparative claims
6. **# Implementation Guidelines:**
	- **MUST** follow Step-by-step instructions (generate code/config as described)
	- **MUST** Follow precisely during implementation
	- **RECOMMENDED** Include practical examples, code snippets, configuration files
	- **RECOMMENDED** Provide exact version numbers and dependency specifications
	- **RECOMMENDED** Include setup scripts, commands, or automation tools
	- May contain troubleshooting guides and validation steps
	- Define naming conventions for all relevant artifacts (classes, methods, files, packages, etc.)
	- Specify coding standards and style requirements where applicable
7. **# Additional Recommendations:**
	- **MUST** Reference relevant documentation, tools, or resources
	- Best practices and related suggestions (apply if relevant)
	- May be applied based on specific implementation context
	- Include optimization techniques and advanced configurations
	
8. **# Success Criteria:**
	- Every compilation and coding process **MUST** successfully pass these criteria.
	- Criteria that **MUST** all be met before considering the ADR successfully implemented.
	- **MUST** Include validation methods and expected results in table format
	- **MUST** Provide validation scripts or commands where applicable

---

## ADR Catalog

All ADRs are organized into four categories by folder. Each ADR follows the 8-section template defined above.

### 01 — Software Environment Setup

Foundation tooling, runtime, containerization, and framework decisions.

| ADR | Title | Scope |
|:----|:------|:------|
| 02 | Java Runtime Environment | JDK 21 (Eclipse Temurin 21.0.4), SDKMAN management |
| 03 | Build Automation | Maven 3.9.8, Maven Wrapper, dependency version properties |
| 04 | Toolchain Version Management | Exact version pinning, `setup.sh` validation script |
| 05 | Containerization Infrastructure | Docker 24.0+, Docker Compose 2.24+ |
| 06 | Development Environment Automation | Makefile, `setup.sh`, Docker Compose services (ClickHouse, Jaeger, Prometheus, Grafana) |
| 07 | Containerized Development Environment | Graduated approach: Dev Services → Compose → Testcontainers, `host.docker.internal` |
| 08 | Application Framework (Quarkus) | Quarkus 3.32.2, RESTEasy Reactive, Mutiny, BOM import |
| 13a | ClickHouse Infrastructure Setup | ClickHouse Docker Compose, JDBC 0.6.3, Tabix Web UI, application config |

### 02 — Foundation Setup

Architecture, module structure, cross-cutting concerns, and platform-level patterns.

| ADR | Title | Scope |
|:----|:------|:------|
| 09 | Multi-Module Maven Architecture | Business/External/API/Library/Container module types, forbidden god libraries, platform library catalog |
| 10 | Software Architecture (Hexagonal) | Ports & Adapters, ArchUnit enforcement, domain→port→adapter dependency direction |
| 11 | Package Structure Organization | Standardized package layout (domain/application/infrastructure), Jandex indexing, naming conventions |
| 15 | Application Configuration Management | Comprehensive `application.properties`, profile-based config (%dev/%test/%prod), env vars |
| 16 | Structured Logging Strategy | JSON logging, MDC correlation (traceId, spanId, correlationId), sensitive data redaction |
| 17 | Exception Handling (Reactive) | Layered exception hierarchy (DomainException/InfrastructureException), ErrorContext, Mutiny error patterns, circuit breakers |
| 18 | Security Implementation (OWASP) | JWT RS256, RBAC, input validation/sanitization, AES-256-GCM encryption, security audit logging |
| 19 | Audit Logging Strategy | Structured async audit logging, Javers diffs, dedicated audit logger, AuditLog aggregate |
| 20 | Observability Strategy | Three pillars (Metrics/Logs/Traces), unified correlation, Docker Compose observability stack |
| 21 | Distributed Tracing | OpenTelemetry, Jaeger, @WithSpan, manual span creation, sampling strategies |
| 22 | Application Monitoring & Observability | Grafana dashboards, Prometheus scraping, custom metrics, alerting, datasource UID configuration |
| 23 | REST API Error Handling | RFC 9457 Problem Details, ProblemDetail model, reactive exception mapper, HttpStatusMapper |
| 25 | Internationalization Strategy | Error codes (DOMAIN-CATEGORY-SEQUENCE), resource bundles, reactive locale context, MessageResolver |
| 26 | Message Externalization | Complete string externalization, LocalizedLogger, MessageKeys constants, configurable default locale |

### 03 — Software Development Rules

Coding standards, testing, data handling, API documentation, and generation templates.

| ADR | Title | Scope |
|:----|:------|:------|
| 12 | Code Quality Standards | Google Java Style, Spotless, Checkstyle, PMD, SpotBugs/FindSecBugs |
| 13b | ClickHouse Coding Standards | Schema design, MergeTree engines, batch inserts, reactive Mutiny adapters, parameterized queries |
| 14 | Data Migration Strategy | Flyway migrations, `V{version}__{description}.sql` naming, quarkus-flyway extension |
| 24 | Data Transfer & Validation | DTO patterns, Bean Validation 3.0, Java records for responses, MessageKeys, custom validators |
| 27 | API Documentation Localization | OpenAPI localization, internationalized API docs for non-English markets |
| 28 | Testcontainers Integration | Integration testing with ClickHouse Testcontainers, `QuarkusTestResourceLifecycleManager` |
| 29 | Code Generation Templates | Implementation consistency templates for code generation |
| 30 | Business Requirement Format | BRD standard (`brd/` directory, `BR-NNN-descriptive-name.md`), Gherkin scenarios, RFC 2119 rules, cross-cutting concerns checklist |

### 04 — Build Setup

Production artifact generation and deployment packaging.

| ADR | Title | Scope |
|:----|:------|:------|
| 31 | Production Build Artifacts | Coordinated build outputs: application binary, DB migrations, observability dashboards, API docs, error inventory |


## BRD Directory

Business Requirement Documents are stored in `brd/` per ADR-30.

| BRD | Title |
|:----|:------|
| BR-000 | Platform Operational Foundation |

---

## Folder Organization

```
00-adr-structure-summary.md          ← This file (template + catalog)
01-software-environment-setup/       ← Runtime, tooling, containers, framework
02-foundation-setup/                 ← Architecture, modules, cross-cutting concerns
03-software-development-rules/       ← Coding standards, testing, data, APIs
04-build-setup/                      ← Production build artifacts
brd/                                 ← Business Requirement Documents
```
