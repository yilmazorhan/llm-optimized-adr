# Issue: Development Environment Automation

New developers joining the project face manual, error-prone setup of build tools, infrastructure services, and runtime dependencies. Without standardized automation, environment inconsistencies cause "works on my machine" failures and slow onboarding.

# Decision:

Use Maven Wrapper (`mvnw`) and Docker Compose for automated, reproducible development environment setup.

- Maven Wrapper **MUST** be committed at project root to pin the Maven version and eliminate local Maven installation requirements.
- Docker Compose **MUST** manage all infrastructure services: ClickHouse, Jaeger, Prometheus, Grafana.
- A `Makefile` **MUST** provide standardized commands for build, test, run, and infrastructure lifecycle.
- A `setup.sh` script **MUST** validate prerequisites and bootstrap the environment in a single step.
- Multi-module Maven structure **MUST** use dependency management via a parent POM.

# Ownership

| Role | Person | Competencies |
|------|--------|-------------|
| **Responsible** | DevOps / Platform Engineer | Shell scripting for cross-platform setup automation (macOS/Linux), Makefile design for build/test/run targets, Docker Compose service orchestration and health checks, Maven Wrapper integration, developer onboarding workflow optimization |
| **Approver** | Tech Lead / Software Architect | Development workflow standardization decisions, prerequisite toolchain trade-offs, multi-module Maven structure governance, infrastructure service selection (ClickHouse, Jaeger, Prometheus, Grafana) |

# Constraints:

- **MUST NOT** require any tools beyond Java 21 (Eclipse Temurin 21.0.4), Docker 24.0+, and Docker Compose 2.24+.
- **MUST NOT** assume a specific OS; scripts **MUST** work on macOS and Linux.
- Scripts and Makefiles **MUST** support both `docker compose` (plugin) and `docker-compose` (standalone) command variants.
- **MUST NOT** store secrets or credentials in setup scripts or Makefile.
- Infrastructure services **MUST** be verified as ready before declaring setup complete.

# Alternatives:

1. **Gradle Wrapper + Testcontainers-only**: Gradle provides similar wrapper functionality and Testcontainers can replace Docker Compose for service management. Rejected because: Quarkus official documentation and extensions primarily target Maven (https://quarkus.io/guides/maven-tooling). Testcontainers starts/stops containers per test run, adding 10–30 seconds of container startup overhead per cycle versus persistent Docker Compose services.


# Rationale:

Maven Wrapper is the Quarkus-recommended build tool wrapper (https://quarkus.io/guides/maven-tooling#maven-wrapper). It guarantees all developers use the identical Maven version without manual installation. Docker Compose provides declarative, persistent infrastructure services that survive across multiple dev/test cycles, avoiding repeated container startup overhead. A Makefile abstracts multi-step commands into single targets, reducing onboarding from 4–5 manual commands to `make setup`. This combination requires only JDK and Docker as prerequisites — both standard in Java development environments.

# Implementation Guidelines:

## Prerequisites:
- **Java 21 (Eclipse Temurin 21.0.4 via SDKMAN)** (ADR-02)
- **Docker 24.0+ with Docker Compose plugin 2.24+ OR standalone docker-compose** (ADR-05)

## Critical Implementation Requirements:

### Docker Compose Command Compatibility
Scripts and Makefiles **MUST** support both Docker Compose command variants. Failing to support both **SHALL** cause environment-specific failures:
- `docker compose` (Docker Desktop plugin, newer)
- `docker-compose` (standalone installation, legacy but still common)

**Makefile Pattern:**
```makefile
# MUST auto-detect docker compose command
DOCKER_COMPOSE := $(shell docker compose version >/dev/null 2>&1 && echo "docker compose" || echo "docker-compose")

start:
	$(DOCKER_COMPOSE) -f local-env/docker-compose.yml up -d
```

**Shell Script Pattern:**
```bash
# MUST check for both docker compose variants
if docker compose version &> /dev/null; then
    COMPOSE_CMD="docker compose"
elif docker-compose version &> /dev/null; then
    COMPOSE_CMD="docker-compose"
else
    echo "❌ Docker Compose not available"
    exit 1
fi
```

### Multi-Module Quarkus Dev Mode
When running `quarkus:dev` on a specific module in a multi-module project, the `-am` (also-make) flag **MUST** be included to build dependencies first. Omitting this flag **SHALL** cause dependency resolution failures:

```bash
# CORRECT: MUST use -am flag to build dependencies first
./mvnw quarkus:dev -pl container-app -am

# INCORRECT: SHALL fail if dependencies not in local repository
./mvnw quarkus:dev -pl container-app
```

**Makefile dev target:**
```makefile
dev: ## Start development server with live reload
	# MUST include -am flag for multi-module projects
	./mvnw quarkus:dev -pl container-app -am
```

## Setup Process:

**1. Clone and Build:**
```bash
git clone <repository-url>
cd copilot-quarkus
./mvnw clean install  # Install all modules to local repository
```

**2. Start Infrastructure:**
```bash
make start  # Start ClickHouse, Jaeger, Prometheus, Grafana (auto-detects Docker Compose variant)
```

**3. Run Application:**
```bash
make dev  # Start in development mode with live reload (includes -am for multi-module)
```

### Setup Script (`setup.sh`):
```bash
#!/usr/bin/env bash
set -e

echo "🚀 Copilot Quarkus Demo - Environment Setup"
echo "==========================================="

# Check prerequisites
echo "📋 Checking prerequisites..."
if ! command -v java &> /dev/null; then
    echo "❌ Java not found. Please install Eclipse Temurin 21.0.4 via SDKMAN."
    exit 1
fi
JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
if [[ "$JAVA_VERSION" != 21.0.4* ]]; then
    echo "❌ Java version $JAVA_VERSION detected. Required: 21.0.4."
    exit 1
fi

if ! command -v docker &> /dev/null; then
    echo "❌ Docker not found. Please install Docker 24.0 or newer."
    exit 1
fi

# MUST check for both docker compose variants
if docker compose version &> /dev/null; then
    COMPOSE_CMD="docker compose"
elif docker-compose version &> /dev/null; then
    COMPOSE_CMD="docker-compose"
else
    echo "❌ Docker Compose not available. Please install Docker Compose 2.24+."
    exit 1
fi

echo "✅ Prerequisites check passed"

# Build all modules
echo "🔨 Building all modules..."
./mvnw clean install -q

# Start infrastructure
echo "🐳 Starting infrastructure services..."
$COMPOSE_CMD up -d

# Wait for services to be ready
echo "⏳ Waiting for services to start..."
sleep 10

# Verify services
echo "🔍 Verifying services..."
if $COMPOSE_CMD ps | grep -q "Up"; then
    echo "✅ Infrastructure services are running"
else
    echo "❌ Some services failed to start"
    $COMPOSE_CMD logs
    exit 1
fi

echo ""
echo "✨ Setup complete! Next steps:"
echo "  1. Start development: make dev"
echo "  2. View application: http://localhost:8080"
echo "  3. View Swagger UI: http://localhost:8080/q/swagger-ui"
echo "  4. View monitoring: http://localhost:3000 (admin/admin)"
```

### Makefile:
```makefile
# MUST auto-detect docker compose command
DOCKER_COMPOSE := $(shell docker compose version >/dev/null 2>&1 && echo "docker compose" || echo "docker-compose")

.PHONY: help setup dev build test package clean start stop logs status demo build-native docker-build install-libs install-business verify

# Default target
help: ## Show this help message
	@echo "Copilot Quarkus Demo - Available Commands"
	@echo "========================================"
	@awk 'BEGIN {FS = ":.*##"} /^[a-zA-Z_-]+:.*##/ {printf "  %-15s %s\n", $$1, $$2}' $(MAKEFILE_LIST)

setup: ## Setup development environment (run once)
	@echo "🚀 Setting up development environment..."
	./setup.sh

dev: ## Start development server with live reload
	@echo "🔥 Starting development server..."
	# MUST include -am flag for multi-module projects
	./mvnw quarkus:dev -pl container-app -am

build: ## Build all modules
	@echo "🔨 Building all modules..."
	./mvnw clean install

test: ## Run all tests
	@echo "🧪 Running tests..."
	./mvnw test

package: ## Package application
	@echo "📦 Packaging application..."
	./mvnw clean package -pl container-app

clean: ## Clean all build artifacts
	@echo "🧹 Cleaning build artifacts..."
	./mvnw clean

start: ## Start infrastructure services
	@echo "🐳 Starting infrastructure services..."
	$(DOCKER_COMPOSE) up -d

stop: ## Stop infrastructure services
	@echo "🛑 Stopping infrastructure services..."
	$(DOCKER_COMPOSE) down

logs: ## Show infrastructure logs
	@echo "📋 Infrastructure logs..."
	$(DOCKER_COMPOSE) logs -f

status: ## Show service status
	@echo "📊 Service status..."
	$(DOCKER_COMPOSE) ps
	@echo ""
	@echo "🌐 Access points:"
	@echo "  Application:     http://localhost:8080"
	@echo "  Swagger UI:      http://localhost:8080/q/swagger-ui"
	@echo "  Health:          http://localhost:8080/q/health"
	@echo "  Metrics:         http://localhost:8080/q/metrics"
	@echo "  Jaeger:          http://localhost:16686"
	@echo "  Prometheus:      http://localhost:9090"
	@echo "  Grafana:         http://localhost:3000"

demo: ## Run rate limiting demonstration
	@echo "🔒 Running rate limiting demo..."
	./demo-rate-limiting.sh

# Advanced targets
build-native: ## Build native executable (requires GraalVM)
	@echo "⚡ Building native executable..."
	./mvnw clean package -pl container-app -Pnative

docker-build: ## Build Docker image
	@echo "🐳 Building Docker image..."
	./mvnw clean package -pl container-app
	docker build -f Dockerfile -t copilot-quarkus-app .

install-libs: ## Install library modules only
	@echo "📚 Installing library modules..."
	./mvnw clean install -pl validation-library,security-library,metrics-library,configuration-library,exception-library,logging-library

install-business: ## Install business modules
	@echo "🏢 Installing business modules..."
	./mvnw clean install -pl user-business,user-external,user-api

verify: ## Run full verification with quality checks
	@echo "✅ Running full verification..."
	./mvnw clean verify
```

## Maven Wrapper Configuration:
- **Maven Version**: Managed by Maven Wrapper (consistent across environments)
- **Java Version**: Java 21 (ADR-02)
- **Quarkus Version**: 3.32.2 (ADR-08)
- **Multi-module Structure**: 10 focused modules (ADR-09)

## Docker Compose Services:
```yaml
services:
  clickhouse:      # Database - ClickHouse 24.x
    ports: 8123:8123, 9000:9000
  jaeger:         # Distributed Tracing - Jaeger all-in-one
    ports: 16686:16686, 4317:4317, 4318:4318
  prometheus:     # Metrics Collection - Prometheus 2.54.1
    ports: 9090:9090
  grafana:        # Monitoring Dashboard - Grafana 11.1.0
    ports: 3000:3000
```

## Quick Commands:
```bash
# Development workflow
make setup              # One-time setup
make dev               # Start development server
make build             # Build all modules
make test              # Run tests
make package           # Package application

# Infrastructure management
make start             # Start services
make stop              # Stop services
make status            # Check service status
make logs              # View logs

# Advanced operations
make build-native      # Build GraalVM native image
make docker-build      # Build Docker image
make verify            # Full quality checks
make demo              # Run rate limiting demo
```

## Access Points:
- **Application**: http://localhost:8080
- **Swagger UI**: http://localhost:8080/q/swagger-ui
- **Health Checks**: http://localhost:8080/q/health
- **Metrics**: http://localhost:8080/q/metrics
- **Jaeger UI**: http://localhost:16686
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000 (admin/admin)

# Additional Recommendations:

- **RECOMMENDED** Use `make setup` for one-time environment initialization and `make dev` for daily development workflow.
- **RECOMMENDED** Include a comprehensive README.md with prerequisites, setup instructions, troubleshooting, and access points.
- **RECOMMENDED** Provide demo scripts (e.g., `demo-rate-limiting.sh`) to showcase application capabilities.
- **RECOMMENDED** Support native compilation via `make build-native` for production GraalVM builds.
- **MUST** Reference ADR-02 (Java Runtime), ADR-05 (Containerization), ADR-08 (Quarkus Framework), and ADR-09 (Multi-Module Structure) as related decisions.
- See Maven Wrapper documentation: https://maven.apache.org/wrapper/
- See Docker Compose documentation: https://docs.docker.com/compose/

# Success Criteria:

> **Completion Gate**: The following criteria **MUST** all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Makefile exists | `ls Makefile` | File present at project root |
| Make targets available | `make help` | Lists build, test, dev, start, stop targets |
| Docker Compose auto-detection | `make start` | Works with both `docker compose` and `docker-compose` |
| Maven Wrapper functional | `./mvnw --version` | Prints Maven version, exit code 0 |
| Full build succeeds | `make build` | `./mvnw clean install` completes with exit code 0 |
| Dev mode starts | `make dev` | Application starts with `-am` flag on port 8080 |
| Infrastructure services run | `make status` | ClickHouse, Jaeger, Prometheus, Grafana all running |
| Setup script works | `./setup.sh` | Validates prerequisites, builds, starts services, exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Development Environment Automation..."

# Check Makefile
if [ -f "Makefile" ]; then
  echo "✅ Makefile exists"
  grep -E "^[a-zA-Z_-]+:" Makefile | head -10 | sed 's/:.*//g' | while read target; do
    echo "   - Target: $target"
  done
else
  echo "❌ Makefile not found"
  exit 1
fi

# Verify Maven Wrapper JAR exists (MUST NOT be excluded by .gitignore)
if [[ ! -f ".mvn/wrapper/maven-wrapper.jar" ]]; then
  echo "❌ maven-wrapper.jar missing — must be present for wrapper to function"
  exit 1
fi
echo "✅ Maven Wrapper JAR present"

# Verify Docker Compose auto-detection in Makefile
if grep -q "DOCKER_COMPOSE" Makefile; then
  echo "✅ Docker Compose auto-detection configured"
else
  echo "❌ Docker Compose auto-detection missing in Makefile"
  exit 1
fi

# Verify -am flag in dev target
if grep -A2 "^dev:" Makefile | grep -q "\-am"; then
  echo "✅ Dev target includes -am flag"
else
  echo "❌ Dev target missing -am flag for multi-module support"
  exit 1
fi

# Verify Maven wrapper works
./mvnw --version > /dev/null && echo "✅ Maven wrapper functional"

# Test compilation
./mvnw clean compile -q && echo "✅ Development build successful"

echo "✅ All Development Environment Automation criteria met"
```
