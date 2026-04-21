# Issue: Production Build Artifacts and Outputs Strategy

Cloud-native application deployments require multiple coordinated artifacts beyond the application binary. Without standardized production build outputs, teams face inconsistent deployments, missing database migrations, undocumented observability dashboards, incomplete API documentation, and lack of error inventory for operations teams. This leads to deployment failures, extended incident response times, and poor developer experience.

**Impact**: Missing or inconsistent build artifacts cause deployment delays, database schema drift, observability gaps, and difficulty troubleshooting production issues.

# Decision:

Implement a comprehensive production build pipeline that generates **five mandatory artifacts** from a single build process. All production builds **MUST** produce these artifacts in a reproducible, automated manner integrated with the CI/CD pipeline.

**Required Production Build Outputs**:

| # | Artifact | Format | Purpose |
|---|----------|--------|---------|
| 1 | **Application Docker Image** | OCI Image | Production-ready application container with all tests passed |
| 2 | **Flyway Migration Docker Image** | OCI Image | Database schema migration container with rollback capability |
| 3 | **Grafana Dashboard JSON** | JSON | Observability dashboard definitions for Grafana provisioning |
| 4 | **Error Inventory Document** | Markdown | Comprehensive error code registry with remediation guidance |
| 5 | **OpenAPI Specification** | YAML | API contract documentation following OpenAPI 3.x specification |

# Constraints:

**Build Process Constraints**:
- All five artifacts **MUST** be generated from a single build invocation
- Build **MUST NOT** succeed if any test fails (unit, integration, or architecture tests)
- Build **MUST** complete within 15 minutes for CI/CD efficiency
- All artifacts **MUST** be versioned consistently with the application version
- Build **MUST** be reproducible with identical inputs producing identical outputs

**Artifact-Specific Constraints**:
- Application image **MUST** pass all security scans before production deployment
- Migration image **MUST** support both forward migration and rollback operations
- Grafana dashboard **MUST** reference datasource UIDs matching provisioning configuration (ADR-22)
- Error inventory **MUST** include all error codes defined in the codebase (ADR-23)
- OpenAPI specification **MUST** be valid according to OpenAPI 3.x schema validation

**Infrastructure Constraints**:
- All Docker images **MUST** use multi-stage builds for minimal image size
- Images **MUST** run as non-root users for security compliance
- Artifacts **MUST** be publishable to artifact repositories (Docker registry, object storage)

# Alternatives:

**Manual Artifact Generation**: The Docker documentation on multi-stage builds (https://docs.docker.com/build/building/multi-stage/) describes how build pipelines chain stages to produce outputs. Manual artifact generation requires invoking each stage independently — `docker build`, `./mvnw package`, script execution — with no mechanism to enforce that all outputs derive from the same Git commit SHA. The Flyway documentation (https://documentation.red-gate.com/flyway/flyway-concepts/migrations) requires migration versions to match application versions; manual processes cannot enforce this constraint atomically.

**Single Monolithic Image**: The OCI Image Specification (https://github.com/opencontainers/image-spec/blob/main/spec.md) defines images as layered filesystems. Bundling application runtime, Flyway CLI, Grafana export tools, and OpenAPI generators in one image produces a layer sum exceeding 1 GB (Eclipse Temurin 21 JRE base ~200 MB + Maven ~300 MB + Flyway ~100 MB + application). The CIS Docker Benchmark v1.6.0 (https://www.cisecurity.org/benchmark/docker) Section 4.1 recommends "one process per container" — multi-purpose images violate this by requiring multiple entrypoints and increasing the attack surface.

# Rationale:

The Quarkus container-image extension documentation (https://quarkus.io/guides/container-image) builds OCI images as part of `mvn package`, binding image tagging to the Maven `project.version` property — this ensures the application image version matches the POM version in the same build invocation. The Flyway versioned migration naming convention (https://documentation.red-gate.com/flyway/flyway-concepts/migrations#versioned-migrations) uses `V{version}__description.sql`; packaging migrations into a dedicated Docker image tagged with the same `project.version` guarantees schema and application versions are synchronized. The Grafana Dashboard JSON Model documentation (https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/) defines an exportable JSON structure with `uid` and `version` fields that can be version-controlled alongside application code. The OpenAPI 3.0.3 specification (https://spec.openapis.org/oas/v3.0.3) Section 4.8.1 defines the `info.version` field, which SmallRye OpenAPI populates from `quarkus.application.version` — exporting during the same build ensures the API contract version matches the deployed artifact.

# Implementation Guidelines:

## Prerequisites

Before generating production artifacts, the following services **MUST** be running:

| Artifact | Required Services | Start Command |
|----------|-------------------|---------------|
| Application Docker Image | None (build-time only) | N/A |
| Flyway Migration Image | None (build-time only) | N/A |
| Grafana Dashboard Export | Grafana (for live export) | `make start` or `docker-compose up -d grafana` |
| Error Inventory | None (static analysis) | N/A |
| OpenAPI Specification | Application (runtime export) | `make start` or `./mvnw quarkus:dev` |

**Important**: The `make openapi` and `make dashboards` targets require running services to export live configurations. For CI/CD pipelines, use the following workflow:

1. Start infrastructure services: `docker-compose up -d clickhouse jaeger prometheus grafana`
2. Start the application: `./mvnw quarkus:dev -Dquarkus.http.port=8080` (background)
3. Wait for health check: `curl -f http://localhost:8080/health`
4. Generate artifacts: `make artifacts`
5. Stop services: `docker-compose down`

**Alternative (Offline Mode)**: For environments where running services is not possible:
- **OpenAPI**: Use Maven plugin generation: `./mvnw compile quarkus:generate-code -pl container-app`
- **Dashboards**: Copy from provisioned directory: `monitoring/grafana/dashboards/`

## 1. Application Docker Image (with Tests)

The primary application container **MUST** be built after successful test execution.

**Maven Build Configuration**:
```bash
# Build application with all tests
./mvnw clean verify -pl container-app -am

# Build Docker image using Quarkus container extension
./mvnw package -pl container-app -am \
  -Dquarkus.container-image.build=true \
  -Dquarkus.container-image.name=copilot-quarkus-app \
  -Dquarkus.container-image.tag=${VERSION}
```

**Dockerfile (Multi-stage)**:
```dockerfile
# Stage 1: Build
FROM maven:3.9.8-eclipse-temurin-21-alpine AS build
WORKDIR /app
COPY . .
RUN ./mvnw clean package -DskipTests -pl container-app -am

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Security: Run as non-root user
RUN addgroup -S quarkus && adduser -S quarkus -G quarkus
USER quarkus

COPY --from=build /app/container-app/target/quarkus-app /app

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -q --spider http://localhost:8080/q/health/live || exit 1

ENTRYPOINT ["java", "-jar", "/app/quarkus-run.jar"]
```

**Quarkus Container Image Configuration** (`application.properties`):
```properties
# Container Image Configuration
quarkus.container-image.build=true
quarkus.container-image.group=copilot
quarkus.container-image.name=copilot-quarkus-app
quarkus.container-image.tag=${quarkus.application.version}
quarkus.container-image.additional-tags=latest

# Use JIB for efficient layered images
quarkus.container-image.builder=jib
quarkus.jib.base-jvm-image=eclipse-temurin:21-jre-alpine

# Labels for traceability
quarkus.jib.labels."org.opencontainers.image.source"=https://github.com/your-org/copilot-quarkus
quarkus.jib.labels."org.opencontainers.image.version"=${quarkus.application.version}
quarkus.jib.labels."org.opencontainers.image.created"=${maven.build.timestamp}
```

## 2. Flyway Migration Docker Image

A dedicated container for database schema migrations with rollback capability.

**Migration Image Structure**:
```
flyway-migrations/
├── Dockerfile
├── conf/
│   └── flyway.conf
├── sql/
│   ├── V1.0.0__initial_schema.sql
│   ├── V1.0.1__add_audit_events.sql
│   ├── V1.1.0__add_user_indexes.sql
│   └── U1.1.0__rollback_user_indexes.sql  # Undo migration
└── scripts/
    ├── migrate.sh
    └── rollback.sh
```

**Flyway Migration Dockerfile**:
```dockerfile
FROM flyway/flyway:10-alpine

# Copy migration scripts
COPY sql/ /flyway/sql/
COPY conf/flyway.conf /flyway/conf/flyway.conf
COPY scripts/ /flyway/scripts/

# Make scripts executable
RUN chmod +x /flyway/scripts/*.sh

# Environment variables for ClickHouse connection
ENV FLYWAY_URL=jdbc:clickhouse://localhost:8123/demo
ENV FLYWAY_USER=demo
ENV FLYWAY_PASSWORD=demo
ENV FLYWAY_SCHEMAS=demo
ENV FLYWAY_LOCATIONS=filesystem:/flyway/sql
ENV FLYWAY_BASELINE_ON_MIGRATE=true
ENV FLYWAY_VALIDATE_ON_MIGRATE=true

# Default command: migrate
ENTRYPOINT ["/flyway/scripts/migrate.sh"]
```

**Migration Script** (`scripts/migrate.sh`):
```bash
#!/bin/bash
set -e

echo "🔄 Starting Flyway migration..."
echo "   Target: ${FLYWAY_URL}"
echo "   Schema: ${FLYWAY_SCHEMAS}"

# Validate migrations before applying
flyway validate

# Apply migrations
flyway migrate

# Show migration status
flyway info

echo "✅ Migration completed successfully"
```

**Rollback Script** (`scripts/rollback.sh`):
```bash
#!/bin/bash
set -e

TARGET_VERSION=${1:-}

if [ -z "$TARGET_VERSION" ]; then
    echo "❌ Usage: rollback.sh <target_version>"
    echo "   Example: rollback.sh 1.0.0"
    exit 1
fi

echo "⚠️  Rolling back to version: ${TARGET_VERSION}"
echo "   Target: ${FLYWAY_URL}"

# Show current state
flyway info

# Perform undo migrations (requires Flyway Teams or manual undo scripts)
# For community edition, use targeted undo scripts
flyway undo -target="${TARGET_VERSION}"

# Verify rollback
flyway info

echo "✅ Rollback to ${TARGET_VERSION} completed"
```

**ClickHouse-Specific Flyway Configuration** (`conf/flyway.conf`):
```properties
# Flyway Configuration for ClickHouse
flyway.driver=com.clickhouse.jdbc.ClickHouseDriver
flyway.url=${FLYWAY_URL}
flyway.user=${FLYWAY_USER}
flyway.password=${FLYWAY_PASSWORD}
flyway.schemas=${FLYWAY_SCHEMAS}
flyway.locations=filesystem:/flyway/sql

# Migration behavior
flyway.baselineOnMigrate=true
flyway.validateOnMigrate=true
flyway.outOfOrder=false
flyway.cleanDisabled=true

# ClickHouse-specific settings
flyway.placeholders.clickhouse_cluster=default
flyway.placeholders.replication_factor=1
```

**Sample ClickHouse Migration** (`sql/V1.0.0__initial_schema.sql`):
```sql
-- V1.0.0: Initial schema for audit events
-- ClickHouse MergeTree engine optimized for time-series analytical queries

CREATE TABLE IF NOT EXISTS audit_events (
    event_id UUID DEFAULT generateUUIDv4(),
    timestamp DateTime64(3) DEFAULT now64(3),
    
    -- Service context
    service_name LowCardinality(String),
    service_version LowCardinality(String),
    
    -- Event classification
    event_name LowCardinality(String),
    event_category LowCardinality(String),
    outcome LowCardinality(Enum8('SUCCESS' = 1, 'FAILURE' = 2)),
    
    -- Correlation
    correlation_id String,
    trace_id String,
    span_id String,
    
    -- Actor information
    actor_type LowCardinality(Enum8('USER' = 1, 'SYSTEM' = 2, 'API_KEY' = 3)),
    actor_id String,
    
    -- Resource information
    resource_type LowCardinality(String),
    resource_id String,
    
    -- Event data
    event_data String CODEC(ZSTD(3))
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (service_name, event_name, timestamp)
TTL timestamp + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;
```

## 3. Grafana Dashboard JSON Export

Export Grafana dashboard definitions for Infrastructure-as-Code provisioning.

**Dashboard Export Script** (`scripts/export-dashboards.sh`):
```bash
#!/bin/bash
set -e

OUTPUT_DIR="${1:-build/dashboards}"
GRAFANA_URL="${GRAFANA_URL:-http://localhost:3000}"
GRAFANA_API_KEY="${GRAFANA_API_KEY:-}"

mkdir -p "$OUTPUT_DIR"

echo "📊 Exporting Grafana dashboards..."

# List of dashboards to export (by UID)
DASHBOARDS=(
    "application-overview"
    "controller-performance"
    "quarkus-framework"
    "security-monitoring"
    "business-intelligence"
)

for dashboard in "${DASHBOARDS[@]}"; do
    echo "   Exporting: $dashboard"
    
    curl -s -H "Authorization: Bearer ${GRAFANA_API_KEY}" \
        "${GRAFANA_URL}/api/dashboards/uid/${dashboard}" \
        | jq '.dashboard' > "${OUTPUT_DIR}/${dashboard}.json"
done

# Also copy static provisioning dashboards
if [ -d "monitoring/grafana/dashboards" ]; then
    cp monitoring/grafana/dashboards/*.json "$OUTPUT_DIR/" 2>/dev/null || true
fi

echo "✅ Dashboards exported to: $OUTPUT_DIR"
ls -la "$OUTPUT_DIR"
```

**Dashboard Validation Script** (`scripts/validate-dashboards.sh`):
```bash
#!/bin/bash
set -e

DASHBOARD_DIR="${1:-build/dashboards}"

echo "🔍 Validating Grafana dashboards..."

for file in "$DASHBOARD_DIR"/*.json; do
    if [ -f "$file" ]; then
        filename=$(basename "$file")
        
        # Validate JSON syntax
        if ! jq empty "$file" 2>/dev/null; then
            echo "❌ Invalid JSON: $filename"
            exit 1
        fi
        
        # Validate required fields
        if ! jq -e '.title' "$file" > /dev/null 2>&1; then
            echo "❌ Missing 'title' field: $filename"
            exit 1
        fi
        
        # Validate datasource UIDs match provisioning
        DATASOURCE_UIDS=$(jq -r '.. | .datasource?.uid? // empty' "$file" 2>/dev/null | sort -u)
        for uid in $DATASOURCE_UIDS; do
            if [[ "$uid" != "prometheus" && "$uid" != "jaeger" && "$uid" != "loki" ]]; then
                echo "⚠️  Non-standard datasource UID in $filename: $uid"
            fi
        done
        
        echo "✅ Valid: $filename"
    fi
done

echo "✅ All dashboards validated"
```

## 4. Error Inventory Document (Markdown)

Generate comprehensive error code documentation from codebase analysis.

**Error Inventory Generator** (`scripts/generate-error-inventory.sh`):
```bash
#!/bin/bash
set -e

OUTPUT_FILE="${1:-build/ERROR_INVENTORY.md}"
SOURCE_DIRS="container-app/src exception-library/src platform-api/src"

echo "📋 Generating Error Inventory..."

cat > "$OUTPUT_FILE" << 'EOF'
# Error Inventory

> **Generated**: $(date -u +"%Y-%m-%dT%H:%M:%SZ")  
> **Version**: ${PROJECT_VERSION}

This document provides a comprehensive inventory of all error codes generated by the application, following the RFC 9457 Problem Details specification (ADR-23).

## Error Code Format

Error codes follow the pattern: `[DOMAIN]-[CATEGORY]-[SEQUENCE]`

- **DOMAIN**: Business domain (e.g., `USER`, `ORDER`, `AUTH`)
- **CATEGORY**: Error category (e.g., `VALIDATION`, `BUSINESS`, `INFRASTRUCTURE`)
- **SEQUENCE**: Numeric sequence (e.g., `001`, `002`)

---

## Domain Errors

### User Domain

| Code | HTTP Status | Title | Description | Retryable | Remediation |
|------|-------------|-------|-------------|-----------|-------------|
EOF

# Extract error codes from HttpStatusMapper and exception classes
grep -rh "DOM-[0-9]\{3\}" $SOURCE_DIRS 2>/dev/null | \
    grep -oE '"DOM-[0-9]{3}"' | sort -u | \
    while read code; do
        echo "| $code | - | - | - | No | - |"
    done >> "$OUTPUT_FILE"

cat >> "$OUTPUT_FILE" << 'EOF'

### Infrastructure Errors

| Code | HTTP Status | Title | Description | Retryable | Remediation |
|------|-------------|-------|-------------|-----------|-------------|
EOF

grep -rh "INF-[0-9]\{3\}" $SOURCE_DIRS 2>/dev/null | \
    grep -oE '"INF-[0-9]{3}"' | sort -u | \
    while read code; do
        echo "| $code | - | - | - | Yes | - |"
    done >> "$OUTPUT_FILE"

cat >> "$OUTPUT_FILE" << 'EOF'

---

## HTTP Status Code Mapping

| Status Code | Meaning | When Used |
|-------------|---------|-----------|
| 400 | Bad Request | Validation errors, malformed input |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Business rule violation (e.g., duplicate email) |
| 422 | Unprocessable Entity | Valid syntax but invalid business state |
| 500 | Internal Server Error | Unexpected server error |
| 502 | Bad Gateway | External service failure |
| 503 | Service Unavailable | Database or dependency unavailable |
| 504 | Gateway Timeout | External service timeout |

---

## Troubleshooting Guide

### Common Error Patterns

1. **DOM-001 (Email Already Exists)**
   - **Cause**: Attempting to create a user with an email that already exists
   - **Resolution**: Use a different email address or retrieve the existing user

2. **DOM-002 (Entity Not Found)**
   - **Cause**: Requested resource ID does not exist in the system
   - **Resolution**: Verify the ID and check if the resource was deleted

3. **INF-001 (External Service Error)**
   - **Cause**: Downstream service is unavailable or returned an error
   - **Resolution**: Retry after 30 seconds; check service health

### Correlation ID Usage

All error responses include a `correlationId` header that can be used to trace the request through logs:

```bash
# Search logs by correlation ID
grep "correlation_id=CORR-12345" /var/log/app/*.log
```

EOF

echo "✅ Error inventory generated: $OUTPUT_FILE"
```

**Enhanced Error Inventory Template** (`build/ERROR_INVENTORY.md`):
```markdown
# Error Inventory

> **Generated**: 2026-01-27T10:00:00Z  
> **Application Version**: 1.0.0  
> **Specification**: RFC 9457 (Problem Details for HTTP APIs)

This document provides a comprehensive inventory of all error codes generated by the application.

---

## Domain Errors (DOM-XXX)

Domain errors represent business rule violations and are typically not retryable.

| Code | HTTP Status | Title | Description | Retryable |
|------|-------------|-------|-------------|-----------|
| DOM-001 | 409 | Email Already Exists | The email address is already registered | No |
| DOM-002 | 404 | Entity Not Found | The requested resource does not exist | No |
| DOM-003 | 400 | Validation Error | Input data failed validation | No |
| DOM-004 | 422 | Invalid Business State | Operation not allowed in current state | No |
| DOM-005 | 403 | Authorization Failure | Insufficient permissions | No |

### Remediation Steps

#### DOM-001: Email Already Exists
- Use a different email address
- If you already have an account, use the login endpoint
- Contact support if you believe this is an error

#### DOM-002: Entity Not Found
- Verify the resource ID is correct
- Check if the resource was recently deleted
- Use the list endpoint to find available resources

---

## Infrastructure Errors (INF-XXX)

Infrastructure errors represent technical failures and may be retryable.

| Code | HTTP Status | Title | Description | Retryable | Retry-After |
|------|-------------|-------|-------------|-----------|-------------|
| INF-001 | 502 | External Service Error | Downstream service unavailable | Yes | 30s |
| INF-002 | 503 | Database Connection Error | Database connection failed | Yes | 30s |
| INF-003 | 504 | Gateway Timeout | External service timeout | Yes | 60s |
| INF-004 | 500 | Configuration Error | Application misconfiguration | No | - |
| INF-005 | 503 | Circuit Breaker Open | Service temporarily unavailable | Yes | 30s |

---

## Example Error Response

```json
{
  "type": "https://api.copilot-quarkus.com/problems/DOM-001",
  "title": "Email Already Exists",
  "status": 409,
  "detail": "The email address 'user@example.com' is already registered",
  "instance": "CORR-ABC123",
  "errorCode": "DOM-001",
  "correlationId": "CORR-ABC123",
  "timestamp": "2026-01-27T10:00:00.000Z",
  "retryable": false,
  "traceId": "abc123def456",
  "spanId": "789xyz"
}
```

---

## Support Information

For unresolved issues, contact support with:
- Error code
- Correlation ID
- Timestamp
- Request details (sanitized)
```

## 5. OpenAPI Specification (YAML)

Generate OpenAPI specification from application annotations.

**OpenAPI Generation Configuration** (`application.properties`):
```properties
# OpenAPI Configuration
quarkus.smallrye-openapi.path=/openapi
quarkus.smallrye-openapi.info-title=Copilot Quarkus API
quarkus.smallrye-openapi.info-version=${quarkus.application.version}
quarkus.smallrye-openapi.info-description=Cloud-native microservices API
quarkus.smallrye-openapi.info-contact-name=API Support
quarkus.smallrye-openapi.info-contact-email=api-support@example.com
quarkus.smallrye-openapi.info-license-name=Apache 2.0
quarkus.smallrye-openapi.info-license-url=https://www.apache.org/licenses/LICENSE-2.0

# Security schemes
quarkus.smallrye-openapi.security-scheme=jwt
quarkus.smallrye-openapi.security-scheme-name=bearerAuth
quarkus.smallrye-openapi.security-scheme-description=JWT Bearer token authentication
```

**OpenAPI Export Script** (`scripts/export-openapi.sh`):
```bash
#!/bin/bash
set -e

OUTPUT_FILE="${1:-build/openapi.yaml}"
APP_URL="${APP_URL:-http://localhost:8080}"

echo "📄 Generating OpenAPI specification..."

# Start application in test mode if not running
if ! curl -sf "${APP_URL}/q/health/ready" > /dev/null 2>&1; then
    echo "   Starting application for OpenAPI generation..."
    ./mvnw quarkus:dev -pl container-app -am &
    APP_PID=$!
    
    # Wait for application to be ready
    for i in {1..60}; do
        if curl -sf "${APP_URL}/q/health/ready" > /dev/null 2>&1; then
            break
        fi
        sleep 1
    done
fi

# Export OpenAPI specification
curl -sf "${APP_URL}/q/openapi" > "$OUTPUT_FILE"

# Validate OpenAPI specification
if command -v openapi-generator-cli &> /dev/null; then
    openapi-generator-cli validate -i "$OUTPUT_FILE"
fi

# Stop application if we started it
if [ -n "$APP_PID" ]; then
    kill $APP_PID 2>/dev/null || true
fi

echo "✅ OpenAPI specification generated: $OUTPUT_FILE"

# Show summary
echo ""
echo "📊 API Summary:"
yq '.info.title' "$OUTPUT_FILE" 2>/dev/null || jq -r '.info.title' "$OUTPUT_FILE"
yq '.info.version' "$OUTPUT_FILE" 2>/dev/null || jq -r '.info.version' "$OUTPUT_FILE"
echo "   Paths: $(yq '.paths | keys | length' "$OUTPUT_FILE" 2>/dev/null || jq '.paths | keys | length' "$OUTPUT_FILE")"
```

**Maven OpenAPI Generation** (alternative):
```xml
<plugin>
    <groupId>io.smallrye</groupId>
    <artifactId>smallrye-open-api-maven-plugin</artifactId>
    <version>4.2.4</version>
    <executions>
        <execution>
            <id>generate-openapi</id>
            <phase>package</phase>
            <goals>
                <goal>generate-schema</goal>
            </goals>
            <configuration>
                <configProperties>
                    <mp.openapi.scan.packages>com.copilot.quarkus.infrastructure.web</mp.openapi.scan.packages>
                </configProperties>
                <outputDirectory>${project.build.directory}</outputDirectory>
                <schemaFilename>openapi</schemaFilename>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Unified Build Pipeline

**Makefile Targets**:
```makefile
.PHONY: build-production artifacts docker-app docker-migrations dashboards error-inventory openapi

VERSION ?= $(shell ./mvnw help:evaluate -Dexpression=project.version -q -DforceStdout)

# Main production build target
build-production: test artifacts ## Build all production artifacts
	@echo "✅ Production build complete. Artifacts in build/"

# Run all tests (prerequisite for production build)
test: ## Run all tests
	@echo "🧪 Running all tests..."
	./mvnw clean verify -pl container-app -am

# Generate all artifacts
artifacts: docker-app docker-migrations dashboards error-inventory openapi ## Generate all build artifacts
	@echo "📦 All artifacts generated"

# 1. Application Docker Image
docker-app: ## Build application Docker image
	@echo "🐳 Building application Docker image..."
	./mvnw package -pl container-app -am -DskipTests \
		-Dquarkus.container-image.build=true \
		-Dquarkus.container-image.tag=$(VERSION)

# 2. Flyway Migration Docker Image
docker-migrations: ## Build migration Docker image
	@echo "🗃️ Building migration Docker image..."
	docker build -t copilot-quarkus-migrations:$(VERSION) \
		-f flyway-migrations/Dockerfile \
		flyway-migrations/

# 3. Grafana Dashboards
dashboards: ## Export Grafana dashboard JSON
	@echo "📊 Exporting Grafana dashboards..."
	mkdir -p build/dashboards
	./scripts/export-dashboards.sh build/dashboards
	./scripts/validate-dashboards.sh build/dashboards

# 4. Error Inventory
error-inventory: ## Generate error inventory markdown
	@echo "📋 Generating error inventory..."
	mkdir -p build
	./scripts/generate-error-inventory.sh build/ERROR_INVENTORY.md

# 5. OpenAPI Specification
openapi: ## Generate OpenAPI specification
	@echo "📄 Generating OpenAPI specification..."
	mkdir -p build
	./scripts/export-openapi.sh build/openapi.yaml

# Publish artifacts
publish: build-production ## Publish all artifacts to registries
	@echo "📤 Publishing artifacts..."
	docker push copilot-quarkus-app:$(VERSION)
	docker push copilot-quarkus-migrations:$(VERSION)
	# Copy static artifacts to object storage
	aws s3 cp build/ s3://artifacts-bucket/$(VERSION)/ --recursive
```

**CI/CD Pipeline Example** (`.github/workflows/build.yml`):
```yaml
name: Production Build

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Run Tests
        run: ./mvnw clean verify -pl container-app -am
      
      - name: Build Application Image
        run: |
          ./mvnw package -pl container-app -am -DskipTests \
            -Dquarkus.container-image.build=true \
            -Dquarkus.container-image.push=true \
            -Dquarkus.container-image.registry=${{ secrets.REGISTRY }}
      
      - name: Build Migration Image
        run: |
          docker build -t ${{ secrets.REGISTRY }}/copilot-migrations:${{ github.ref_name }} \
            -f flyway-migrations/Dockerfile flyway-migrations/
          docker push ${{ secrets.REGISTRY }}/copilot-migrations:${{ github.ref_name }}
      
      - name: Generate Artifacts
        run: |
          mkdir -p build
          make dashboards error-inventory openapi
      
      - name: Upload Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: production-artifacts
          path: build/
```

## Validation Commands:

```bash
# Full production build with all artifacts
make build-production

# Verify all artifacts exist
ls -la build/
# Expected: dashboards/, ERROR_INVENTORY.md, openapi.yaml

# Verify Docker images
docker images | grep copilot-quarkus

# Test migration container
docker run --rm copilot-quarkus-migrations:latest \
  -url=jdbc:clickhouse://host:8123/demo \
  -user=demo -password=demo \
  info

# Validate OpenAPI specification
openapi-generator-cli validate -i build/openapi.yaml

# Test application container health
docker run -d -p 8080:8080 --name test-app copilot-quarkus-app:latest
sleep 10
curl http://localhost:8080/q/health
docker stop test-app && docker rm test-app
```

# Additional Recommendations:

- Use Quarkus JIB extension for efficient layered Docker images without requiring a local Docker daemon
- Implement semantic versioning for all artifacts to maintain clear release history
- Add OCI image labels (source, version, created) for container traceability
- Validate Grafana dashboard JSON against datasource UID conventions before export
- Include rollback migration scripts (`U{version}__description.sql`) for every forward migration

**References**:
- Quarkus Container Image Guide: https://quarkus.io/guides/container-image
- Flyway Migrations Documentation: https://documentation.red-gate.com/flyway/flyway-concepts/migrations
- Grafana Dashboard JSON Model: https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/
- OpenAPI 3.0.3 Specification: https://spec.openapis.org/oas/v3.0.3
- OCI Image Specification: https://github.com/opencontainers/image-spec/blob/main/spec.md
- SmallRye OpenAPI Maven Plugin: https://github.com/smallrye/smallrye-open-api/tree/main/tools/maven-plugin

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Tests Pass | `./mvnw clean verify -pl container-app -am` | Exit code 0 |
| App Image Config | `grep -r "container-image" */src/main/resources/application.properties 2>/dev/null \| head -3` | Quarkus container-image properties configured |
| Migration Dockerfile | `find . -path "*/flyway-migrations/Dockerfile" -type f \| head -1` | Flyway migration Dockerfile exists |
| Grafana Dashboards | `find . -path "*/grafana/dashboards/*.json" -type f \| head -5` | JSON dashboard files present |
| Error Inventory Script | `find . -name "generate-error-inventory.sh" -type f \| head -1` | Error inventory generator exists |
| OpenAPI Extension | `grep -rl "smallrye-openapi" pom.xml */pom.xml 2>/dev/null \| head -3` | SmallRye OpenAPI dependency configured |
| Makefile Targets | `grep -E "build-production\|artifacts" Makefile 2>/dev/null \| head -3` | Production build targets present |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
echo "Validating Production Build Artifacts..."

FAIL=0

# Check tests pass
if ./mvnw clean verify -pl container-app -am -q 2>/dev/null; then
  echo "✅ All tests pass"
else
  echo "❌ Tests failed"
  FAIL=1
fi

# Check Quarkus container-image configuration
FOUND=0
for f in $(find . -name "application.properties" -path "*/src/main/resources/*" 2>/dev/null); do
  if grep -ql "container-image" "$f" 2>/dev/null; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Quarkus container-image extension configured"
else
  echo "❌ Quarkus container-image properties not found in application.properties"
  FAIL=1
fi

# Check Flyway migration Dockerfile
FOUND=0
if find . -path "*/flyway-migrations/Dockerfile" -type f 2>/dev/null | grep -q .; then
  FOUND=1
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Flyway migration Dockerfile exists"
else
  echo "❌ Flyway migration Dockerfile not found (expected flyway-migrations/Dockerfile)"
  FAIL=1
fi

# Check Grafana dashboard JSON files
FOUND=0
for f in $(find . -path "*/grafana/dashboards/*.json" -type f 2>/dev/null); do
  if [ -f "$f" ]; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  DASHBOARD_COUNT=$(find . -path "*/grafana/dashboards/*.json" -type f 2>/dev/null | wc -l | tr -d ' ')
  echo "✅ Grafana dashboards found: $DASHBOARD_COUNT"
else
  echo "❌ No Grafana dashboard JSON files found"
  FAIL=1
fi

# Check error inventory generation script
FOUND=0
if find . -name "generate-error-inventory.sh" -type f 2>/dev/null | grep -q .; then
  FOUND=1
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Error inventory generation script exists"
else
  echo "❌ Error inventory generation script not found (expected scripts/generate-error-inventory.sh)"
  FAIL=1
fi

# Check SmallRye OpenAPI extension
FOUND=0
for f in pom.xml $(find . -maxdepth 2 -name "pom.xml" 2>/dev/null); do
  if [ -f "$f" ] && grep -ql "smallrye-openapi" "$f" 2>/dev/null; then
    FOUND=1
    break
  fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ SmallRye OpenAPI extension configured"
else
  echo "❌ SmallRye OpenAPI extension not found in any pom.xml"
  FAIL=1
fi

# Check Makefile production targets
FOUND=0
if [ -f "Makefile" ] && grep -ql "build-production\|artifacts" Makefile 2>/dev/null; then
  FOUND=1
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Makefile has production build targets"
else
  echo "❌ Makefile missing or lacks build-production/artifacts targets"
  FAIL=1
fi

# Full build verification
if ./mvnw clean verify -q 2>/dev/null; then
  echo "✅ Full build verification passed"
else
  echo "❌ Full build verification failed"
  FAIL=1
fi

if [ "$FAIL" -ne 0 ]; then
  echo "❌ Production Build Artifacts validation failed"
  exit 1
fi

echo "✅ All Production Build Artifacts criteria met"
```
