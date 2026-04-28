# Issue: Toolchain Version Management for Development Consistency

Unpinned tool versions create build failures, "works on my machine" issues, environment drift, increased debugging time, and reduced team productivity.

This affects JDK, Maven, Docker, and other development tools across local development, CI/CD pipelines, and production builds.

# Ownership

| Role | Person | Competencies |
|------|--------|-------------|
| **Responsible** | DevOps / Platform Engineer | CI/CD pipeline design, Docker and container runtime internals, shell scripting for environment validation, cross-platform toolchain distribution (SDKMAN, Maven Wrapper), infrastructure-as-code |
| **Approver** | Tech Lead / Software Architect | Cross-team tooling decisions, build reproducibility trade-offs, JDK distribution landscape evaluation, Maven dependency resolution internals, onboarding friction assessment |

# Decision:

All critical development tools **MUST** have explicitly pinned versions enforced across all environments (local, CI/CD, production).

Version compliance **MUST** be automated and verified before development/deployment activities.

**Requirements:**
- All tools **MUST** use exact version pinning (version ranges **MUST NOT** be used)
- Enforcement **MUST** be automated via scripts/CI/CD checks (manual checks **MUST NOT** be relied upon)
- Selected tools **MUST** support cross-platform compatibility
- All pinned versions **MUST** be clearly documented
- Validation **MUST** fast-fail with clear error messages

# Constraints:

**Technical:**
- Projects **MUST** use exact tool versions specified by the project configuration
- Flexible version ranges that allow drift **MUST NOT** be used
- Automated verification **MUST** be implemented (manual checks **MUST NOT** be the sole verification method)
- Selected tools **MUST** be available cross-platform

**Operational:**
- Version pinning **MUST NOT** create significant onboarding friction
- Version updates **MUST** be coordinated across the team with documented migration steps
- CI/CD pipelines **MUST** include version checks as blocking validation gates

**Business:**
- Selected tool versions **MUST** have long-term support
- All tools **MUST** comply with organizational licensing policies
- Tool licensing and support costs **SHOULD** be considered during version selection

# Alternatives:

**Flexible Version Ranges** (e.g., Maven 3.9.x): Rejected. Teams **MUST NOT** use flexible version ranges.

Maven 3.9.x range allows automatic adoption of patch releases. Maven release notes document breaking behavioral changes between patch versions (e.g., Maven 3.9.2 changed dependency resolution order per [MNG-7734](https://issues.apache.org/jira/browse/MNG-7734)). Similarly, JDK patch releases have introduced behavioral changes in garbage collection and security providers (documented in Oracle JDK Release Notes). Flexible ranges make builds non-reproducible, violating the Maven reproducible builds specification ([Maven Reproducible Builds](https://maven.apache.org/guides/mini/guide-reproducible-builds.html)).

**Docker-based Development Environment as Primary**: Rejected. Docker-based development **MUST NOT** be used as the primary local development approach.

Docker Desktop on macOS uses a Linux VM with file system translation (VirtioFS/gRPC-FUSE), which introduces measured I/O latency. Docker's official documentation acknowledges macOS file sharing performance limitations ([Docker Desktop for Mac - File Sharing](https://docs.docker.com/desktop/settings/mac/#file-sharing)). Quarkus dev mode relies on hot-reload with frequent file I/O, making this latency impactful for developer experience. Docker-based development **MAY** be retained as a fallback option (see Additional Recommendations).

# Rationale:

Explicit version pinning **MUST** be used because it eliminates environment drift by ensuring every developer and CI/CD pipeline uses identical tool versions, removing "works on my machine" issues.

Deterministic reproducible builds are achieved because SDKMAN `.sdkmanrc` ([SDKMAN Configuration](https://sdkman.io/usage#config)) and Maven Wrapper ([Maven Wrapper Plugin](https://maven.apache.org/wrapper/)) lock exact tool versions per project, ensuring identical output across environments. All projects **MUST** use these mechanisms.

Onboarding **SHOULD** be reduced to running `sdk env install` and `./mvnw`, which automatically install the pinned versions without manual version selection.

CI/CD pipelines **MUST** validate the same pinned versions before build execution to prevent deployment failures caused by version mismatches, as documented in the Success Criteria validation script.

This approach uses built-in tooling (SDKMAN `.sdkmanrc`, Maven Wrapper `.mvn/wrapper/`) and **MUST NOT** require additional infrastructure or third-party version management tools.

# Implementation Guidelines:

## Mandatory Version Pinning:

1. **JDK Version Management**:
   - Projects **MUST** use Eclipse Temurin JDK 21.0.4 (exact version pinned)
   - SDKMAN **MUST** be configured with `.sdkmanrc` file in project root:
     ```bash
     # .sdkmanrc
     java=21.0.4-tem
     ```
   - Setup scripts **MUST** verify exact version:
     ```bash
     JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
     if [[ "$JAVA_VERSION" != "21.0.4"* ]]; then
         echo "❌ Java version $JAVA_VERSION detected. Required: 21.0.4"
         exit 1
     fi
     ```

2. **Maven Version Control**:
   - Projects **MUST** use Maven 3.9.8 enforced via Maven Wrapper
   - Use maven to create wrappers  ```mvn wrapper:wrapper -Dmaven=3.9.8```
   - Version **MUST** be pinned in `.mvn/wrapper/maven-wrapper.properties`:
     ```properties
     distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.8/apache-maven-3.9.8-bin.zip
     ```
   - Maven version **MUST** be validated via wrapper:
     ```bash
     MAVEN_VERSION=$(./mvnw -v | grep 'Apache Maven' | awk '{print $3}')
     if [[ "$MAVEN_VERSION" != "3.9.8"* ]]; then
         echo "❌ Maven version $MAVEN_VERSION detected. Required: 3.9.8"
         exit 1
     fi
     ```

3. **Container Tool Versions**:
   - Docker **MUST** be version 24.0+ (minimum for compatibility)
   - Docker Compose **MUST** be version 2.24+ (minimum for compatibility)
   - Tested versions **MUST** be documented in project documentation

4. **Setup Script Requirements**:
   - A `setup.sh` (or equivalent) **MUST** be created that validates all tool versions
   - The script **MUST** fail fast with clear error messages for version mismatches
   - The script **MUST** provide installation guidance for incorrect versions
   - The script **MUST** be executable on all supported platforms

5. **CI/CD Integration**:
   - Version validation **MUST** execute before any build or test steps
   - Builds **MUST** fail immediately if version requirements are not met
   - CI/CD pipelines **MUST** use identical toolchain versions as local development

## Documentation Requirements:
- All required versions **MUST** be documented in README.md prerequisites section
- Version change history **MUST** be maintained in ADR updates
- A troubleshooting guide for common version issues **SHOULD** be provided
- Documentation **MUST** be updated when tool versions change

## Example Implementation
```bash
#!/usr/bin/env bash

# Quarkus Demo Project Setup Script
# This script demonstrates the complete development workflow

set -e

echo "🚀 Setting up Quarkus Demo Project..."


# Check prerequisites
echo "📋 Checking prerequisites..."

# Enforce SDKMAN usage if available
if command -v sdk &> /dev/null; then
    echo "ℹ️  SDKMAN detected. Using .sdkmanrc for JDK and Maven version pinning."
    sdk env install || true
    sdk env
fi

# Check Java version
if ! command -v java &> /dev/null; then
    echo "❌ Java is not installed. Please install JDK 21.0.4 (Eclipse Temurin recommended)."
    exit 1
fi
JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
if [[ "$JAVA_VERSION" != 21.0.4* ]]; then
    echo "❌ Java version $JAVA_VERSION detected. Please use JDK 21.0.4 as pinned in .sdkmanrc."
    exit 1
fi
echo "✅ Java version: $JAVA_VERSION"

# Check Maven version
if ! ./mvnw -v &> /dev/null; then
    echo "❌ Maven Wrapper is not working. Please check mvnw permissions."
    exit 1
fi
MAVEN_VERSION=$(./mvnw -v | grep 'Apache Maven' | awk '{print $3}')
if [[ "$MAVEN_VERSION" != 3.9.8* ]]; then
    echo "❌ Maven version $MAVEN_VERSION detected. Please use Maven 3.9.8 as pinned in Maven Wrapper."
    exit 1
fi
echo "✅ Maven version: $MAVEN_VERSION (via Maven Wrapper)"


# Check Docker
if ! command -v docker &> /dev/null; then
    echo "❌ Docker is not installed. Please install Docker 24.0+ (tested)."
    echo "ℹ️  For platform-specific installation instructions and troubleshooting, see the 'Docker Containerization Infrastructure' ADR (ADR-05)."
    exit 1
fi
DOCKER_VERSION=$(docker --version | awk '{print $3}' | sed 's/,//')
echo "✅ Docker version: $DOCKER_VERSION"

# Check Docker Compose plugin
if ! docker compose version &> /dev/null; then
    echo "❌ Docker Compose plugin is not available. Please install Docker Compose 2.24+ (tested)."
    exit 1
fi
# Prefer human-readable flag for Docker Compose version where supported
DOCKER_COMPOSE_VERSION=$(docker compose version --short 2>/dev/null || docker compose version | awk '{print $3}' | sed 's/,//')
echo "✅ Docker Compose version: $DOCKER_COMPOSE_VERSION"

# Make Maven wrapper executable
chmod +x mvnw

echo "🔧 Building the application..."
./mvnw clean compile

echo "🧪 Running tests..."
./mvnw test

echo "📦 Packaging the application..."
./mvnw package -DskipTests

echo "🐳 Starting infrastructure with Docker Compose..."
docker compose up -d clickhouse

echo "⏳ Waiting for ClickHouse to be ready..."
until docker compose exec clickhouse clickhouse-client --query "SELECT 1" > /dev/null 2>&1; do
  echo "ClickHouse is not ready yet. Waiting..."
  sleep 2
done

echo "✅ ClickHouse is ready!"

echo "🎯 Starting Quarkus in development mode..."
echo "📝 Note: The application will be available at http://localhost:8080"
echo "📊 Swagger UI will be available at http://localhost:8080/swagger-ui"
echo "🔍 Health checks available at http://localhost:8080/q/health"
echo "📈 Metrics available at http://localhost:8080/q/metrics"

echo ""
echo "🏃‍♂️ To start the application in development mode, run:"
echo "   ./mvnw quarkus:dev"
echo ""
echo "🐳 To start the complete stack with monitoring, run:"
echo "   docker compose up"
echo ""
echo "🧪 To run integration tests with Testcontainers, run:"
echo "   ./mvnw verify"
echo ""
echo "🏗️ To build native image, run:"
echo "   ./mvnw package -Pnative"
echo ""

echo "✨ Setup complete! Happy coding!"

```

# Additional Recommendations:

## Reference Documentation:
Implementers **MUST** consult the following official documentation:
- **SDKMAN**: [https://sdkman.io/](https://sdkman.io/) — `.sdkmanrc` configuration: [https://sdkman.io/usage#config](https://sdkman.io/usage#config)
- **Maven Wrapper**: [https://maven.apache.org/wrapper/](https://maven.apache.org/wrapper/) — Plugin docs: [https://maven.apache.org/plugins/maven-wrapper-plugin/](https://maven.apache.org/plugins/maven-wrapper-plugin/)
- **Eclipse Temurin JDK 21**: [https://adoptium.net/temurin/releases/](https://adoptium.net/temurin/releases/)
- **Docker Engine**: [https://docs.docker.com/engine/](https://docs.docker.com/engine/) — Release notes: [https://docs.docker.com/engine/release-notes/](https://docs.docker.com/engine/release-notes/)
- **Docker Compose**: [https://docs.docker.com/compose/](https://docs.docker.com/compose/) — Release notes: [https://docs.docker.com/compose/release-notes/](https://docs.docker.com/compose/release-notes/)

**Example Enhanced Setup Validation**:
```bash
#!/usr/bin/env bash
# Enhanced setup script with comprehensive validation

set -e

echo "🔧 Validating development environment..."

# Function to check tool version
check_version() {
    local tool=$1
    local expected=$2
    local actual=$3
    
    if [[ "$actual" == "$expected"* ]]; then
        echo "✅ $tool version: $actual"
    else
        echo "❌ $tool version mismatch. Expected: $expected, Found: $actual"
        exit 1
    fi
}

# JDK validation
JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
check_version "Java" "21.0.4" "$JAVA_VERSION"

# Maven validation
MAVEN_VERSION=$(./mvnw -v | grep 'Apache Maven' | awk '{print $3}')
check_version "Maven" "3.9.8" "$MAVEN_VERSION"

# Docker validation
DOCKER_VERSION=$(docker --version | awk '{print $3}' | sed 's/,//')
if [[ "$DOCKER_VERSION" < "24.0" ]]; then
    echo "❌ Docker version $DOCKER_VERSION. Required: 24.0+"
    exit 1
fi
echo "✅ Docker version: $DOCKER_VERSION"

echo "✅ All tools validated successfully!"
```

# Success Criteria:

> **LLM Agent Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| .sdkmanrc Present | `test -f .sdkmanrc && echo "exists"` | "exists" |
| Java Pinned | `grep java .sdkmanrc` | `java=21.0.4-tem` |
| Maven Pinned | `cat .mvn/wrapper/maven-wrapper.properties` | Contains `3.9.8` |
| Docker Version | `docker --version` | 24.0+ |
| Docker Compose | `docker compose version` | 2.24+ |
| Setup Script | `test -f setup.sh && echo "exists"` | "exists" |
| Setup Execution | `./setup.sh` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Toolchain Version Management..."

# Check .sdkmanrc
if [[ ! -f ".sdkmanrc" ]]; then
  echo "❌ .sdkmanrc missing"
  exit 1
fi
echo "✅ .sdkmanrc present"

# Check Maven Wrapper JAR exists (MUST NOT be excluded by .gitignore)
if [[ ! -f ".mvn/wrapper/maven-wrapper.jar" ]]; then
  echo "❌ maven-wrapper.jar missing — must be present for wrapper to function"
  exit 1
fi
echo "✅ Maven Wrapper JAR present"

# Verify Java pinning
if ! grep -q "java=21.0.4-tem" .sdkmanrc; then
  echo "❌ Java not pinned to 21.0.4-tem"
  exit 1
fi
echo "✅ Java version pinned"

# Check actual versions match
JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
if [[ "$JAVA_VERSION" != 21.0.4* ]]; then
  echo "❌ Java $JAVA_VERSION does not match pinned version"
  exit 1
fi
echo "✅ Java version matches: $JAVA_VERSION"

MAVEN_VERSION=$(./mvnw -v | grep 'Apache Maven' | awk '{print $3}')
if [[ "$MAVEN_VERSION" != "3.9.8" ]]; then
  echo "❌ Maven $MAVEN_VERSION does not match pinned version"
  exit 1
fi
echo "✅ Maven version matches: $MAVEN_VERSION"

echo "✅ All Toolchain Version Management criteria met"
```
