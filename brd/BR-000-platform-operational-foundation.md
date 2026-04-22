# BR-000: Platform Operational Foundation

## 1. Summary

As a **developer**, I want **a fully configured development environment with all required tools and infrastructure services installed and validated**, so that **I can immediately begin building business features on a reproducible, observable platform without manual environment troubleshooting**.

This BRD defines the step-by-step software installation procedure for establishing the development platform. It covers SDKMAN, JDK 21.0.4, Maven Wrapper 3.9.8, Docker 24.0+, Docker Compose 2.24+, infrastructure services (ClickHouse, Tabix, Jaeger, Prometheus, Grafana), Quarkus 3.32.2 framework bootstrapping, and automated validation. No business domain logic, entities, or feature endpoints SHALL be included.

## 2. Business Rules

- **BR-000-01**: SDKMAN **MUST** be installed as the cross-platform JDK version manager before any JDK installation. (ADR-02, ADR-04)
- **BR-000-02**: JDK 21.0.4 (Eclipse Temurin) **MUST** be installed via SDKMAN and pinned in `.sdkmanrc` with the value `java=21.0.4-tem`. (ADR-02)
- **BR-000-03**: `JAVA_HOME` **MUST** point to the JDK 21.0.4 installation path. `java -version` and `javac -version` **MUST** both output version `21.0.4`. (ADR-02)
- **BR-000-04**: Maven Wrapper 3.9.8 **MUST** be present at the project root (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/`). All build commands **MUST** use `./mvnw`. System-wide Maven installations **MUST NOT** be used for project builds. (ADR-03)
- **BR-000-05**: `.mvn/wrapper/maven-wrapper.properties` **MUST** contain `apache-maven-3.9.8` in its `distributionUrl`. (ADR-03, ADR-04)
- **BR-000-06**: Docker Engine 24.0+ **MUST** be installed. The Docker daemon **MUST** be running and accessible without `sudo` on Linux/macOS. (ADR-05)
- **BR-000-07**: Docker Compose 2.24+ **MUST** be available. Scripts **MUST** support both `docker compose` (plugin) and `docker-compose` (standalone) command variants. (ADR-05, ADR-06)
- **BR-000-08**: Docker **MUST** have a minimum of 4GB memory, 2 CPU cores, and 10GB disk allocated. (ADR-05)
- **BR-000-09**: A `setup.sh` script **MUST** exist at the project root and **MUST** validate all tool versions automatically. The script **MUST** fast-fail with clear error messages on version mismatches. (ADR-04, ADR-06)
- **BR-000-10**: A `Makefile` **MUST** provide standardized commands: `setup`, `dev`, `build`, `test`, `package`, `start`, `stop`, `logs`, `status`, `verify`. The Makefile **MUST** auto-detect the Docker Compose command variant. (ADR-06)
- **BR-000-11**: Docker Compose **MUST** manage five infrastructure services: ClickHouse (ports 8123, 9000), Tabix (port 8124), Jaeger (ports 16686, 4317, 4318), Prometheus (port 9090), and Grafana (port 3000). All services **MUST** include health checks. (ADR-07, ADR-13a)
- **BR-000-12**: ClickHouse 24.x **MUST** be the configured database service with HTTP interface (port 8123) and native TCP interface (port 9000). Tabix Web UI **MUST** be enabled at port 8124. (ADR-13a)
- **BR-000-13**: Quarkus 3.32.2 BOM **MUST** be imported in the parent `pom.xml`. RESTEasy Reactive (`quarkus-rest`) **MUST** be used. Classic JAX-RS (`quarkus-resteasy`) **MUST NOT** be used. (ADR-08)
- **BR-000-14**: All dependency versions **MUST** be defined in the `<properties>` section of the parent `pom.xml` and referenced using property placeholders. Version numbers **MUST NOT** be hardcoded directly in `<dependency>` or `<plugin>` declarations. (ADR-03)
- **BR-000-15**: Prometheus **MUST** use `host.docker.internal:8080` as scrape target when the application runs on the host machine. (ADR-07)
- **BR-000-16**: `./mvnw clean compile` **MUST** complete with exit code 0 after installation. (ADR-03)
- **BR-000-17**: All tool versions **MUST** be exact-pinned. Version ranges **MUST NOT** be used. (ADR-04)

## 3. Acceptance Criteria / Scenarios

```gherkin
Scenario: SDKMAN installation
  Given a development machine with bash or zsh shell
  When I run "curl -s https://get.sdkman.io | bash"
  And I run "source $HOME/.sdkman/bin/sdkman-init.sh"
  Then the command "sdk version" exits with code 0
  And SDKMAN is available in the current shell session

Scenario: JDK 21.0.4 installation via SDKMAN
  Given SDKMAN is installed and initialized
  When I run "sdk install java 21.0.4-tem"
  And I run "sdk default java 21.0.4-tem"
  Then "java -version" output contains "21.0.4"
  And "javac -version" output contains "21.0.4"
  And the file ".sdkmanrc" contains "java=21.0.4-tem"

Scenario: Maven Wrapper validation
  Given the project repository is cloned
  When I run "chmod +x mvnw"
  And I run "./mvnw -v"
  Then the output contains "Apache Maven 3.9.8"
  And files "mvnw", "mvnw.cmd", ".mvn/wrapper/maven-wrapper.properties" exist
  And ".mvn/wrapper/maven-wrapper.properties" contains "apache-maven-3.9.8"

Scenario: Docker Engine installation verification
  Given Docker Desktop or Docker Engine is installed
  When I run "docker --version"
  Then the output shows version 24.0 or higher
  When I run "docker run --rm hello-world"
  Then the container executes successfully
  And Docker daemon is running without errors

Scenario: Docker Compose installation verification
  Given Docker Engine 24.0+ is installed
  When I run "docker compose version" or "docker-compose version"
  Then the output shows version 2.24 or higher
  And at least one of the two command variants succeeds

Scenario: Docker resource allocation
  Given Docker Desktop is configured
  When I check Docker resource settings
  Then memory allocation is at least 4GB
  And CPU allocation is at least 2 cores
  And disk allocation is at least 10GB

Scenario: Project compilation after installation
  Given JDK 21.0.4 and Maven Wrapper 3.9.8 are validated
  When I run "./mvnw clean compile"
  Then the build exits with code 0
  And no compilation errors are reported

Scenario: Setup script validates all prerequisites
  Given all tools are installed
  When I run "./setup.sh"
  Then the script verifies Java version is 21.0.4
  And the script verifies Maven version is 3.9.8
  And the script verifies Docker version is 24.0+
  And the script verifies Docker Compose version is 2.24+
  And the script exits with code 0

Scenario: Infrastructure services startup via Docker Compose
  Given Docker and Docker Compose are installed and running
  When I run "make start" or "docker compose up -d"
  Then ClickHouse is accessible at "http://localhost:8123" and responds to "SELECT 1"
  And Tabix Web UI is accessible at "http://localhost:8124"
  And Jaeger UI is accessible at "http://localhost:16686"
  And Prometheus is accessible at "http://localhost:9090"
  And Grafana is accessible at "http://localhost:3000"
  And all services reach "healthy" state within 2 minutes

Scenario: Infrastructure services clean shutdown
  Given all Docker Compose services are running
  When I run "make stop" or "docker compose down"
  Then all containers stop cleanly
  And no orphan containers remain

Scenario: Quarkus application starts in development mode
  Given all prerequisites are satisfied and infrastructure is running
  When I run "./mvnw quarkus:dev -pl container-app -am"
  Then the application starts in under 2 seconds
  And "GET /q/health" returns {"status":"UP"}
  And "GET /q/metrics" returns Prometheus metrics
  And "GET /q/openapi" returns a valid OpenAPI specification

Scenario: Prometheus scrapes host application
  Given the application runs on the host via "./mvnw quarkus:dev"
  And Prometheus runs in Docker via Docker Compose
  When I check Prometheus targets at "http://localhost:9090/api/v1/targets"
  Then the target "host.docker.internal:8080" shows health "up"

Scenario: Complete installation verification
  Given all installation steps have been completed
  When I run the full verification script
  Then SDKMAN is functional
  And Java 21.0.4 is active
  And Maven Wrapper 3.9.8 is functional
  And Docker 24.0+ is running
  And Docker Compose 2.24+ is available
  And "./mvnw clean compile" exits with code 0
  And all Docker Compose services start and reach healthy state
  And ClickHouse responds to "SELECT 1"
  And the verification script exits with code 0
```

## 4. Cross-Cutting Concerns Checklist

| Concern | ADR Reference(s) | Plan Details | Verified |
|---------|-------------------|--------------|----------|
| Java Runtime | ADR-02 | JDK 21.0.4 (Eclipse Temurin) installed via SDKMAN; pinned in `.sdkmanrc` | TBD |
| Build Automation | ADR-03 | Maven Wrapper 3.9.8 committed; all commands use `./mvnw` | TBD |
| Toolchain Pinning | ADR-04 | `.sdkmanrc` + `.mvn/wrapper/maven-wrapper.properties` pin exact versions; `setup.sh` validates | TBD |
| Containerization | ADR-05 | Docker 24.0+ / Compose 2.24+ installed; both command variants supported | TBD |
| Dev Environment Automation | ADR-06 | `Makefile` with auto-detected Compose command; `setup.sh` validates prerequisites | TBD |
| Containerized Dev Environment | ADR-07 | Docker Compose with ClickHouse, Tabix, Jaeger, Prometheus, Grafana; health checks; `host.docker.internal` for Prometheus | TBD |
| Application Framework | ADR-08 | Quarkus 3.32.2 BOM imported; RESTEasy Reactive (`quarkus-rest`); Mutiny reactive patterns | TBD |
| ClickHouse Infrastructure | ADR-13a | ClickHouse 24.x in Docker Compose (ports 8123, 9000); Tabix at port 8124 | TBD |

## 5. Related ADRs

- ADR-02: Java Runtime Environment — JDK 21.0.4, Eclipse Temurin, SDKMAN
- ADR-03: Build Automation — Maven Wrapper 3.9.8, dependency version properties
- ADR-04: Toolchain Version Management — Exact version pinning, `setup.sh` validation
- ADR-05: Containerization Infrastructure — Docker 24.0+, Docker Compose 2.24+, dual command support
- ADR-06: Development Environment Automation — Makefile, `setup.sh`, Docker Compose services
- ADR-07: Containerized Development Environment — Graduated approach, `host.docker.internal`
- ADR-08: Application Framework Quarkus — Quarkus 3.32.2, RESTEasy Reactive, Mutiny
- ADR-13a: ClickHouse Infrastructure Setup — ClickHouse Docker Compose, JDBC 0.6.3, Tabix Web UI

## 6. Installation Steps

### Step 1: Install SDKMAN (ADR-02, ADR-04)

SDKMAN **MUST** be installed for cross-platform JDK version management.

```bash
# Install SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# Verify installation
sdk version
```

### Step 2: Install JDK 21.0.4 via SDKMAN (ADR-02)

Eclipse Temurin JDK 21.0.4 **MUST** be installed and set as the default.

```bash
# Install JDK 21.0.4 (Eclipse Temurin)
sdk install java 21.0.4-tem

# Set as current and default version
sdk use java 21.0.4-tem
sdk default java 21.0.4-tem

# Verify installation
java -version    # MUST output: openjdk version "21.0.4" ...
javac -version   # MUST output: javac 21.0.4
```

The project root **MUST** contain a `.sdkmanrc` file:
```
java=21.0.4-tem
```

When entering the project directory, run `sdk env install` to automatically activate the pinned version.

### Step 3: Validate Maven Wrapper (ADR-03)

Maven Wrapper **MUST** be present and functional. System-wide Maven **MUST NOT** be used.

```bash
# Make wrapper executable
chmod +x mvnw

# Verify Maven Wrapper version
./mvnw -v
# MUST output: Apache Maven 3.9.8

# Validate basic build functionality
./mvnw clean validate
```

If the Maven Wrapper is missing, create it using a temporary system Maven installation:
```bash
mvn -N wrapper:wrapper -Dmaven=3.9.8
git add mvnw mvnw.cmd .mvn/
```

### Step 4: Install Docker Engine 24.0+ (ADR-05)

```bash
# macOS
brew install --cask docker

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker.io docker-buildx-plugin

# CentOS/RHEL/Fedora
sudo dnf install docker docker-compose-plugin
```

Post-installation on Linux:
```bash
# Add current user to docker group
sudo usermod -aG docker $USER
# Log out and log back in for group change to take effect
```

Configure Docker Desktop resource limits:
- Memory: minimum 4GB (8GB **RECOMMENDED**)
- CPU: minimum 2 cores (4 cores **RECOMMENDED**)
- Disk: minimum 10GB available

```bash
# Verify Docker installation
docker --version          # MUST output: Docker version 24.0.x or higher
docker run --rm hello-world  # MUST execute successfully
```

### Step 5: Verify Docker Compose 2.24+ (ADR-05)

Docker Compose is included with Docker Desktop. On Linux, it **MAY** require separate installation:

```bash
# Linux (if not included with Docker)
sudo apt-get install docker-compose-plugin

# Verify installation (check BOTH variants)
docker compose version    # Plugin variant
docker-compose version    # Standalone variant
# At least one MUST output version 2.24 or higher
```

### Step 6: Clone Repository and Run Setup (ADR-04, ADR-06)

```bash
# Clone the repository
git clone <repository-url>
cd <project-directory>

# Activate SDKMAN pinned versions
sdk env install

# Run the setup script
./setup.sh
```

The `setup.sh` script **MUST** validate:
1. Java version is exactly 21.0.4
2. Maven Wrapper version is exactly 3.9.8
3. Docker version is 24.0+
4. Docker Compose version is 2.24+ (either command variant)

### Step 7: Build the Project (ADR-03, ADR-08)

```bash
# Compile the project
./mvnw clean compile

# Run full verification (compile, test, quality checks)
./mvnw clean verify
```

### Step 8: Start Infrastructure Services (ADR-06, ADR-07, ADR-13a)

```bash
# Start all infrastructure services
make start
# Or directly: docker compose up -d

# Verify services are running
make status
# Or directly: docker compose ps
```

Services and access points after startup:

| Service | Port(s) | Access URL | Health Check |
|---------|---------|------------|--------------|
| ClickHouse | 8123 (HTTP), 9000 (TCP) | http://localhost:8123 | `SELECT 1` query |
| Tabix (ClickHouse Web UI) | 8124 | http://localhost:8124 | HTTP response |
| Jaeger | 16686 (UI), 4317 (OTLP gRPC), 4318 (OTLP HTTP) | http://localhost:16686 | HTTP response |
| Prometheus | 9090 | http://localhost:9090 | HTTP response |
| Grafana | 3000 | http://localhost:3000 | HTTP response (default: admin/admin) |

### Step 9: Verify ClickHouse Connectivity (ADR-13a)

```bash
# Test ClickHouse via CLI
docker compose exec clickhouse clickhouse-client --query "SELECT version()"

# Test ClickHouse via HTTP
curl -s "http://localhost:8123/ping"

# Verify database
docker compose exec clickhouse clickhouse-client --query "SHOW DATABASES"
```

### Step 10: Start Quarkus Application (ADR-08, ADR-06)

```bash
# Start in development mode with live reload (MUST use -am for multi-module)
make dev
# Or directly: ./mvnw quarkus:dev -pl container-app -am
```

Application endpoints after startup:

| Endpoint | URL | Expected Response |
|----------|-----|-------------------|
| Health Check | http://localhost:8080/q/health | `{"status":"UP"}` |
| Metrics | http://localhost:8080/q/metrics | Prometheus format metrics |
| OpenAPI | http://localhost:8080/q/openapi | Valid OpenAPI specification |
| Swagger UI | http://localhost:8080/q/swagger-ui | Swagger UI page |

## 7. Verification

### Full Installation Verification Script

This script **MUST** be executed after completing all installation steps. All checks **MUST** pass.

```bash
#!/usr/bin/env bash
set -e

echo "========================================="
echo " Platform Installation Verification"
echo "========================================="

FAIL=0

# --- 1. SDKMAN ---
if command -v sdk &> /dev/null; then
  echo "PASS: SDKMAN installed ($(sdk version 2>/dev/null | head -1))"
else
  echo "FAIL: SDKMAN not installed"
  FAIL=1
fi

# --- 2. JDK 21.0.4 ---
if command -v java &> /dev/null; then
  JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
  if [[ "$JAVA_VERSION" == 21.0.4* ]]; then
    echo "PASS: Java version $JAVA_VERSION"
  else
    echo "FAIL: Java version $JAVA_VERSION (expected 21.0.4)"
    FAIL=1
  fi
else
  echo "FAIL: Java not installed"
  FAIL=1
fi

# --- 3. .sdkmanrc ---
if [ -f ".sdkmanrc" ] && grep -q "java=21.0.4-tem" .sdkmanrc; then
  echo "PASS: .sdkmanrc pinned to java=21.0.4-tem"
else
  echo "FAIL: .sdkmanrc missing or not pinned to java=21.0.4-tem"
  FAIL=1
fi

# --- 4. Maven Wrapper ---
if [ -f "mvnw" ] && [ -f "mvnw.cmd" ] && [ -d ".mvn/wrapper" ]; then
  MAVEN_VERSION=$(./mvnw -v 2>/dev/null | grep 'Apache Maven' | awk '{print $3}')
  if [[ "$MAVEN_VERSION" == "3.9.8" ]]; then
    echo "PASS: Maven Wrapper version $MAVEN_VERSION"
  else
    echo "FAIL: Maven Wrapper version $MAVEN_VERSION (expected 3.9.8)"
    FAIL=1
  fi
else
  echo "FAIL: Maven Wrapper files missing (mvnw, mvnw.cmd, .mvn/wrapper/)"
  FAIL=1
fi

# --- 5. Maven Wrapper Properties ---
if [ -f ".mvn/wrapper/maven-wrapper.properties" ] && grep -q "apache-maven-3.9.8" .mvn/wrapper/maven-wrapper.properties; then
  echo "PASS: maven-wrapper.properties contains apache-maven-3.9.8"
else
  echo "FAIL: maven-wrapper.properties missing or version mismatch"
  FAIL=1
fi

# --- 6. Docker ---
if command -v docker &> /dev/null; then
  DOCKER_VERSION=$(docker --version | awk '{print $3}' | sed 's/,//')
  DOCKER_MAJOR=$(echo "$DOCKER_VERSION" | cut -d. -f1)
  if [ "$DOCKER_MAJOR" -ge 24 ]; then
    echo "PASS: Docker version $DOCKER_VERSION"
  else
    echo "FAIL: Docker version $DOCKER_VERSION (expected 24.0+)"
    FAIL=1
  fi
else
  echo "FAIL: Docker not installed"
  FAIL=1
fi

# --- 7. Docker daemon running ---
if docker info &> /dev/null; then
  echo "PASS: Docker daemon is running"
else
  echo "FAIL: Docker daemon is not running"
  FAIL=1
fi

# --- 8. Docker Compose ---
if docker compose version &> /dev/null; then
  COMPOSE_VERSION=$(docker compose version --short 2>/dev/null || docker compose version | awk '{print $3}' | sed 's/,//')
  echo "PASS: Docker Compose version $COMPOSE_VERSION (plugin)"
elif docker-compose version &> /dev/null; then
  COMPOSE_VERSION=$(docker-compose version --short 2>/dev/null || docker-compose version | awk '{print $3}' | sed 's/,//')
  echo "PASS: Docker Compose version $COMPOSE_VERSION (standalone)"
else
  echo "FAIL: Docker Compose not available (neither plugin nor standalone)"
  FAIL=1
fi

# --- 9. Compilation ---
echo "Verifying project compilation..."
if ./mvnw clean compile -q 2>/dev/null; then
  echo "PASS: ./mvnw clean compile succeeded"
else
  echo "FAIL: ./mvnw clean compile failed"
  FAIL=1
fi

# --- 10. setup.sh ---
if [ -f "setup.sh" ] && [ -x "setup.sh" ]; then
  echo "PASS: setup.sh exists and is executable"
else
  echo "FAIL: setup.sh missing or not executable"
  FAIL=1
fi

# --- 11. Makefile ---
if [ -f "Makefile" ]; then
  if grep -q "DOCKER_COMPOSE" Makefile; then
    echo "PASS: Makefile exists with Docker Compose auto-detection"
  else
    echo "FAIL: Makefile missing Docker Compose auto-detection"
    FAIL=1
  fi
else
  echo "FAIL: Makefile not found"
  FAIL=1
fi

# --- 12. Docker Compose file ---
COMPOSE_FILE=$(ls docker-compose*.yml docker-compose*.yaml compose.yml compose.yaml 2>/dev/null | head -1)
if [ -n "$COMPOSE_FILE" ]; then
  echo "PASS: Docker Compose file found: $COMPOSE_FILE"
else
  echo "FAIL: No Docker Compose file found"
  FAIL=1
fi

# --- 13. Docker Compose services (if running) ---
if docker compose ps &> /dev/null 2>&1 || docker-compose ps &> /dev/null 2>&1; then
  # Auto-detect compose command
  if docker compose version &> /dev/null; then
    COMPOSE_CMD="docker compose"
  else
    COMPOSE_CMD="docker-compose"
  fi

  # Check ClickHouse
  if docker ps 2>/dev/null | grep -q clickhouse; then
    if docker exec $(docker ps -q -f name=clickhouse | head -1) clickhouse-client --query "SELECT 1" &> /dev/null; then
      echo "PASS: ClickHouse is running and responds to queries"
    else
      echo "FAIL: ClickHouse container running but not responding"
      FAIL=1
    fi
  else
    echo "INFO: ClickHouse container not running (start with 'make start')"
  fi

  # Check Jaeger
  if curl -sf "http://localhost:16686" &> /dev/null; then
    echo "PASS: Jaeger UI accessible at http://localhost:16686"
  else
    echo "INFO: Jaeger not accessible (start with 'make start')"
  fi

  # Check Prometheus
  if curl -sf "http://localhost:9090/-/ready" &> /dev/null; then
    echo "PASS: Prometheus accessible at http://localhost:9090"
  else
    echo "INFO: Prometheus not accessible (start with 'make start')"
  fi

  # Check Grafana
  if curl -sf "http://localhost:3000/api/health" &> /dev/null; then
    echo "PASS: Grafana accessible at http://localhost:3000"
  else
    echo "INFO: Grafana not accessible (start with 'make start')"
  fi

  # Check Tabix
  if curl -sf "http://localhost:8124" &> /dev/null; then
    echo "PASS: Tabix accessible at http://localhost:8124"
  else
    echo "INFO: Tabix not accessible (start with 'make start')"
  fi
fi

# --- Summary ---
echo "========================================="
if [ "$FAIL" -ne 0 ]; then
  echo " RESULT: VERIFICATION FAILED"
  echo " One or more checks did not pass."
  echo "========================================="
  exit 1
fi
echo " RESULT: ALL CHECKS PASSED"
echo " Platform installation is complete."
echo "========================================="
exit 0
```

### Quick Verification Commands

| Check | Command | Expected Result |
|-------|---------|-----------------|
| SDKMAN | `sdk version` | Version output, exit code 0 |
| Java version | `java -version 2>&1 \| head -1` | Contains `21.0.4` |
| Java compiler | `javac -version` | Contains `21.0.4` |
| SDKMAN pin | `cat .sdkmanrc` | `java=21.0.4-tem` |
| Maven Wrapper files | `ls mvnw mvnw.cmd .mvn/wrapper/` | All files exist |
| Maven version | `./mvnw -v \| grep 'Apache Maven'` | `Apache Maven 3.9.8` |
| Maven properties | `grep apache-maven-3.9.8 .mvn/wrapper/maven-wrapper.properties` | Match found |
| Docker version | `docker --version` | Version 24.0+ |
| Docker daemon | `docker info` | No errors |
| Docker test | `docker run --rm hello-world` | Successful execution |
| Docker Compose | `docker compose version \|\| docker-compose version` | Version 2.24+ |
| Compilation | `./mvnw clean compile` | Exit code 0 |
| Setup script | `./setup.sh` | Exit code 0 |
| ClickHouse | `docker compose exec clickhouse clickhouse-client --query "SELECT 1"` | Returns `1` |
| Tabix UI | `curl -sf http://localhost:8124` | HTTP response |
| Jaeger UI | `curl -sf http://localhost:16686` | HTTP response |
| Prometheus | `curl -sf http://localhost:9090/-/ready` | HTTP response |
| Grafana | `curl -sf http://localhost:3000/api/health` | `{"database":"ok"}` |
| Health check | `curl -sf http://localhost:8080/q/health` | `{"status":"UP"}` |
| Metrics | `curl -sf http://localhost:8080/q/metrics \| head -5` | Prometheus metrics |

