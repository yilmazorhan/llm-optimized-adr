# Issue: Containerization Infrastructure Requirements

Development teams need containerization infrastructure for consistent development-to-production environment parity.

Without containerization, teams face environment drift, dependency conflicts, and inconsistent application behavior across environments.

# Decision:

Docker **MUST** be the containerization platform for all development and deployment environments.

- Docker Engine 24.0+ **MUST** be installed on all development machines.
- Docker Compose 2.24+ **MUST** be available for infrastructure service orchestration.
- Scripts and automation **MUST** support both `docker compose` (plugin) and `docker-compose` (standalone) command variants.
- Container resource limits **MUST** be configured: minimum 4GB memory, 2 CPU cores, 10GB disk.
- Custom networks **SHOULD** be used for service discovery between containers.
- Cross-platform support (macOS/Linux) **MUST** be maintained.

# Ownership

| Role | Person | Competencies |
|------|--------|-------------|
| **Responsible** | DevOps / Platform Engineer | Docker Engine and Compose installation/configuration, container networking and resource management, cross-platform container support (macOS/Linux), CI/CD container integration, Dockerfile security hardening |
| **Approver** | Tech Lead / Software Architect | Container platform selection trade-offs (Docker vs. Podman vs. Kubernetes), Quarkus Dev Services and Testcontainers compatibility assessment, infrastructure resource planning, security policy enforcement for container workloads |

# Constraints:

- **MUST NOT** use Docker versions below 24.0 or Docker Compose below 2.24.
- **MUST NOT** run containers with root privileges unless explicitly required by the service.
- **MUST NOT** use untrusted or unverified base images in Dockerfiles.
- **MUST NOT** store secrets or credentials in Docker images or Compose files.
- Container networks **MUST** be isolated; services **MUST NOT** expose ports beyond what is required.
- System resources **MUST NOT** be exceeded: minimum 4GB RAM, 2 CPU cores, 10GB disk available for Docker.
- Access to Docker Hub or a configured private registry **MUST** be available for image pulls.
- Multi-service applications **MUST** handle startup order via `depends_on` with health checks.

# Alternatives:

1. **Podman**: OCI-compatible container runtime with rootless execution by default. Rejected because: Podman's Docker Compose compatibility layer (`podman-compose`) does not support all Compose v2 features — notably `depends_on` with `condition: service_healthy` has inconsistent behavior (https://github.com/containers/podman-compose/issues/529). Quarkus Dev Services explicitly targets Docker via Testcontainers, which requires Docker socket compatibility (https://www.testcontainers.org/supported_docker_environment/).

2. **Kubernetes (minikube/kind)**: Local Kubernetes clusters for development. Rejected because: minikube requires 2GB+ RAM overhead for the cluster itself (https://minikube.sigs.k8s.io/docs/start/), adding to the 4GB already needed for services. Kubernetes manifests add YAML complexity (Deployment, Service, ConfigMap per service) versus a single `docker-compose.yml`. Local development does not require Kubernetes orchestration features (rolling updates, self-healing, auto-scaling).

# Rationale:

Docker is the container runtime targeted by Quarkus Dev Services and Testcontainers (https://www.testcontainers.org/supported_docker_environment/), making it a prerequisite for the testing strategy in ADR-28. Docker Compose provides declarative, file-based service definitions that can be version-controlled and shared across the team, ensuring all developers run identical infrastructure. The Docker Engine 24.0+ requirement ensures BuildKit support for multi-stage builds and improved caching (https://docs.docker.com/build/buildkit/). Supporting both `docker compose` and `docker-compose` variants covers Docker Desktop users and Linux/CI environments where only the standalone binary is installed.

# Implementation Guidelines:

## Installation Requirements:

**Docker Engine (24.0+):**
```bash
# macOS
brew install --cask docker

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker.io docker-buildx-plugin

# CentOS/RHEL/Fedora
sudo dnf install docker docker-compose-plugin

# Windows
# Download Docker Desktop from https://www.docker.com/products/docker-desktop
```

**Docker Compose (2.24+):**
- Included with Docker Desktop
- Linux additional setup:
```bash
sudo apt-get install docker-compose-plugin
```

**Post-Installation:**
```bash
# Add user to docker group (Linux/macOS)
sudo usermod -aG docker $USER
# Logout and login again

# Configure resource limits:
# Memory: 4GB minimum, 8GB recommended
# CPU: 2 cores minimum, 4 cores recommended  
# Disk: 10GB minimum available space
```

## Validation:
```bash
# Verify Docker installation
docker --version
# Expected: Docker version 24.0.x or higher

# Verify Docker Compose installation (check BOTH variants)
docker compose version || docker-compose version
# Expected: Docker Compose version 2.24.x or higher (or 5.x for standalone)

# Test Docker functionality
docker run hello-world
# Expected: Successful container execution

# Verify Docker Compose Build plugin
docker compose build --help || docker-compose build --help
# Expected: Help output for build command
```

## Docker Compose Command Compatibility:

Scripts and automation **MUST** support both Docker Compose command variants for cross-environment compatibility. Failing to support both **SHALL** cause environment-specific failures:

| Variant | Command | Common On |
|---------|---------|-----------|
| Plugin (new) | `docker compose` | Docker Desktop, newer Linux |
| Standalone (legacy) | `docker-compose` | Older Linux, CI/CD systems |

**Auto-detection Pattern for Makefiles (MUST implement):**
```makefile
# MUST auto-detect available docker compose command
DOCKER_COMPOSE := $(shell docker compose version >/dev/null 2>&1 && echo "docker compose" || echo "docker-compose")
```

**Auto-detection Pattern for Shell Scripts (MUST implement):**
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

## Project Setup:
- Create `docker-compose.yml` for development services
- Create `.dockerignore` files for build optimization
- Create `docker-compose.override.yml` for local customizations
- Document service dependencies and startup order

## Development Workflow:
1. Environment startup: `docker compose up -d`
2. Application development: Run locally against containerized services
3. Testing: Use containers for integration testing
4. Cleanup: `docker compose down`

## Validation Script:
```bash
#!/usr/bin/env bash
# Docker infrastructure validation script

set -e

echo "🐳 Validating Docker infrastructure..."

# Check Docker installation
if ! command -v docker &> /dev/null; then
    echo "❌ Docker not found. Please install Docker 24.0+ before proceeding."
    exit 1
fi

# Check Docker version
DOCKER_VERSION=$(docker --version | awk '{print $3}' | sed 's/,//')
if [[ "${DOCKER_VERSION}" < "24.0" ]]; then
    echo "❌ Docker version ${DOCKER_VERSION} is too old. Please upgrade to 24.0+."
    exit 1
fi
echo "✅ Docker version: ${DOCKER_VERSION}"

# Check Docker Compose (MUST check both variants)
if docker compose version &> /dev/null; then
    COMPOSE_CMD="docker compose"
elif docker-compose version &> /dev/null; then
    COMPOSE_CMD="docker-compose"
else
    echo "❌ Docker Compose not found. Please install Docker Compose 2.24+."
    exit 1
fi

COMPOSE_VERSION=$($COMPOSE_CMD version --short)
echo "✅ Docker Compose version: ${COMPOSE_VERSION} (command: ${COMPOSE_CMD})"

# Test Docker functionality
if ! docker run --rm hello-world &> /dev/null; then
    echo "❌ Docker is not functioning properly. Check Docker daemon status."
    exit 1
fi
echo "✅ Docker functionality verified"

echo "🎉 Docker infrastructure validation complete!"
```

# Additional Recommendations:

- **RECOMMENDED** Use multi-stage Docker builds to minimize final image size. See: https://docs.docker.com/build/building/multi-stage/
- **RECOMMENDED** Implement image security scanning with Trivy (https://aquasecurity.github.io/trivy/) or Docker Scout (https://docs.docker.com/scout/).
- **RECOMMENDED** Run containers as non-root users via `USER` directive in Dockerfiles.
- **RECOMMENDED** Use Eclipse Temurin base images (`eclipse-temurin:21-jre-alpine`) for Java services.
- **RECOMMENDED** Add `healthcheck` directives to all service definitions in `docker-compose.yml`.
- **RECOMMENDED** Use volume mounts for hot reloading during development.
- **RECOMMENDED** Implement semantic versioning for container image tags.
- **RECOMMENDED** Run `docker system prune` periodically to reclaim disk space.
- **MUST** Reference ADR-06 (Development Environment Automation) for Makefile and setup script patterns.
- See Docker Compose documentation: https://docs.docker.com/compose/
- See Docker Engine documentation: https://docs.docker.com/engine/

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Docker Installed | `docker --version` | Version 24.0+ |
| Docker Compose | `docker compose version` or `docker-compose version` | Version 2.24+ (either variant) |
| Docker Running | `docker info` | No errors |
| Hello World | `docker run --rm hello-world` | Success |
| Compose Valid | `docker compose config` | No errors |
| Services Start | `docker compose up -d` | All services healthy |
| Services Stop | `docker compose down` | Clean shutdown |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Containerization Infrastructure..."

# Docker installed
if ! command -v docker &> /dev/null; then
  echo "❌ Docker not installed"
  exit 1
fi
echo "✅ Docker: $(docker --version)"

# Docker Compose (check both variants)
if docker compose version &> /dev/null; then
  COMPOSE_CMD="docker compose"
elif docker-compose version &> /dev/null; then
  COMPOSE_CMD="docker-compose"
else
  echo "❌ Docker Compose not available"
  exit 1
fi
echo "✅ Docker Compose: $($COMPOSE_CMD version --short) (command: $COMPOSE_CMD)"

# Test Docker
if ! docker run --rm hello-world &> /dev/null; then
  echo "❌ Docker not functioning"
  exit 1
fi
echo "✅ Docker functional"

# Validate compose file
if [[ -f "docker-compose.yml" ]] || [[ -f "compose.yml" ]]; then
  $COMPOSE_CMD config > /dev/null
  echo "✅ Compose file valid"
  
  $COMPOSE_CMD up -d
  sleep 10
  $COMPOSE_CMD ps
  $COMPOSE_CMD down
  echo "✅ Services lifecycle verified"
fi

echo "✅ All Containerization criteria met"
```
