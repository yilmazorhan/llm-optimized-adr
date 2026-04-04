# Issue: Containerized Development Environment Requirements

Developer environment management for Quarkus applications with multiple external dependencies requires coordinated setup and configuration. Inconsistent environments cause integration failures, configuration drift, and extended onboarding times.

# Decision:

A graduated containerized development environment strategy **SHALL** be implemented using Docker Compose, Testcontainers, and Quarkus Dev Services. Teams **MUST** progress through complexity levels based on development needs.

**Required Specifications**:
- Quarkus Dev Services (**REQUIRED** for basic development)
- Docker Compose for external services (**MUST** for integration)
- Testcontainers for E2E testing (**MUST** for comprehensive testing)
- **Resource Management**: **MUST** implement selective service startup

# Constraints:

- **MUST** have Docker 24.0+ runtime on all development machines (ADR-05).
- **MUST** work on macOS and Linux. Windows **MUST** use WSL2.
- **SHOULD** implement proper container networking with custom Docker networks.
- Container startup **SHOULD** complete within 2 minutes.
- **MUST** maintain fast feedback loops; hot reload latency **MUST NOT** exceed 5 seconds.
- **MUST NOT** require services beyond what is defined in `docker-compose.yml`.

# Alternatives:

1. **Vagrant + Shell Provisioning**: VM-based development environments using Vagrant with shell provisioners. Rejected because: Vagrant VMs require a full guest OS, consuming 1–2 GB RAM per VM versus 50–200 MB per Docker container (https://www.vagrantup.com/docs/providers). VM boot time is 30–60 seconds versus 1–5 seconds for containers. Vagrant does not integrate with Quarkus Dev Services or Testcontainers.

2. **Local JVM + Embedded Services (H2, embedded Redis)**: Run all dependencies as embedded/in-memory services in the JVM. Rejected because: Embedded alternatives do not exist for ClickHouse (the project's primary database per ADR-13). Embedded services cannot replicate production network topology, port bindings, or container-level resource limits. Testcontainers documentation explicitly recommends real containers over embedded alternatives (https://www.testcontainers.org/#why-testcontainers).

# Rationale:

Quarkus Dev Services automatically provisions containers for detected extensions (e.g., ClickHouse) with zero configuration (https://quarkus.io/guides/dev-services). This provides a working environment from the first `./mvnw quarkus:dev` invocation. Docker Compose adds explicit control for services beyond Dev Services scope (Prometheus, Grafana, Jaeger). Testcontainers provides programmatic container lifecycle management during integration tests, ensuring each test run uses isolated, disposable infrastructure (https://www.testcontainers.org/). The graduated approach (Dev Services → Compose → Testcontainers) allows developers to start with minimal setup and add complexity only when needed.

# Implementation Guidelines:

**Mandatory Setup** (**MUST** be completed):

1. **Tool Requirements**:
   - **MUST** install Docker 24.0+ (ADR-05)
   - **MUST** have JDK 21+ and Maven 3.9.8+ (ADR-02, ADR-03)
   - **SHOULD** configure IDE with Docker integration

2. **Graduated Implementation**:
   ```bash
   # Level 1: Quarkus Dev Services only
   ./mvnw quarkus:dev
   
   # Level 2: Add external services
   docker compose up -d database redis
   ./mvnw quarkus:dev
   
   # Level 3: Full integration testing
   ./mvnw verify -Ptestcontainers
   ```

3. **Resource Configuration**:
   ```yaml
   # docker-compose.yml
   services:
     clickhouse:
       image: clickhouse/clickhouse-server:24.3-alpine
       deploy:
         resources:
           limits:
             memory: 2G
   ```

**Validation** (**SHALL** be performed):
- Verify all levels start successfully within 2 minutes
- Confirm hot reload works in development mode
- Test container cleanup: `docker compose down`

## Host-to-Container Communication

When running the application on the host machine (via `make dev` or `./mvnw quarkus:dev`) while infrastructure services run in Docker containers, container services **MUST** use `host.docker.internal` to reach the host application. Using `localhost` **SHALL NOT** work from within a Docker container.

### Prometheus Configuration for Dev Mode

Prometheus target configuration for development **MUST** follow this pattern:
```yaml
# prometheus/prometheus.yml
scrape_configs:
  - job_name: 'copilot-quarkus'
    metrics_path: '/q/metrics'
    static_configs:
      # MUST use host.docker.internal when app runs on host (make dev)
      - targets: ['host.docker.internal:8080']
```

Docker containers **SHALL NOT** reach `localhost:8080` on the host machine. The special DNS name `host.docker.internal` resolves to the host's IP address from within Docker containers (https://docs.docker.com/desktop/networking/#i-want-to-connect-from-a-container-to-a-service-on-the-host).

### Common Patterns

| App Location | Container Target | Use Case |
|--------------|------------------|----------|
| Host (dev mode) | `host.docker.internal:8080` | **MUST** use for `make dev`, `./mvnw quarkus:dev` |
| Docker container | `app:8080` | **SHOULD** use for `docker-compose up` with app service |

### Connectivity Validation

Validation commands **SHOULD** be run to verify connectivity:
```bash
# Verify Prometheus can reach app metrics
docker exec copilot-prometheus wget -q -O- http://host.docker.internal:8080/q/metrics | head -5

# Check Prometheus targets status
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

# Additional Recommendations:

- **RECOMMENDED** Use Docker Compose profiles for selective service startup to optimize resources.
- **RECOMMENDED** Set container memory/CPU limits in `docker-compose.yml` for all services.
- **RECOMMENDED** Add health checks with `healthcheck` directives to all service definitions.
- **RECOMMENDED** Integrate Prometheus/Grafana for development metrics (ADR-20, ADR-22).
- **RECOMMENDED** Use Jaeger for distributed tracing during development (ADR-21).
- **MUST** Reference ADR-05 (Containerization Infrastructure), ADR-06 (Development Environment Automation), and ADR-28 (Testcontainers Integration).
- See Quarkus Dev Services: https://quarkus.io/guides/dev-services
- See Testcontainers: https://www.testcontainers.org/
- See Docker Compose profiles: https://docs.docker.com/compose/profiles/

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| docker-compose.yml Exists | `ls docker-compose*.yml` | File exists |
| Services Defined | `grep "services:" docker-compose.yml` | Services section present |
| Health Checks | `grep "healthcheck" docker-compose.yml` | Health checks configured |
| Volumes Defined | `grep "volumes:" docker-compose.yml` | Persistent volumes |
| Network Defined | `grep "networks:" docker-compose.yml` | Custom network |
| Services Start | `docker compose up -d` or `docker-compose up -d` | All services running |
| Services Healthy | `docker compose ps` or `docker-compose ps` | All services healthy |
| Services Stop | `docker compose down` or `docker-compose down` | Clean shutdown |
| Host-to-Container | `docker exec <prometheus> wget -q -O- http://host.docker.internal:8080/q/metrics` | Metrics returned |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Containerized Development Environment..."

# Auto-detect Docker Compose command (MUST support both variants per ADR-05)
if docker compose version &> /dev/null; then
  COMPOSE_CMD="docker compose"
elif docker-compose version &> /dev/null; then
  COMPOSE_CMD="docker-compose"
else
  echo "❌ Docker Compose not available"
  exit 1
fi
echo "✅ Docker Compose: $($COMPOSE_CMD version --short) (command: $COMPOSE_CMD)"

# Check for docker-compose file
COMPOSE_FILE=$(ls docker-compose*.yml docker-compose*.yaml compose.yml compose.yaml 2>/dev/null | head -1)
if [ -n "$COMPOSE_FILE" ]; then
  echo "✅ Docker Compose file found: $COMPOSE_FILE"
else
  echo "❌ No Docker Compose file found"
  exit 1
fi

# Check for essential sections
for section in services volumes networks; do
  if grep -q "$section:" "$COMPOSE_FILE"; then
    echo "✅ Section '$section' defined"
  else
    echo "❌ Section '$section' not found in $COMPOSE_FILE"
    exit 1
  fi
done

# Check for health checks
if grep -q "healthcheck" "$COMPOSE_FILE"; then
  echo "✅ Health checks configured"
else
  echo "❌ Health checks not configured in $COMPOSE_FILE"
  exit 1
fi

# Start services
$COMPOSE_CMD up -d
sleep 5

# Check service status
$COMPOSE_CMD ps --format "table {{.Name}}\t{{.Status}}" | while read line; do
  echo "   $line"
done

# Verify all services are running
UNHEALTHY=$($COMPOSE_CMD ps | grep -E "(unhealthy|Exit)" | wc -l)
if [ "$UNHEALTHY" -eq 0 ]; then
  echo "✅ All services running/healthy"
else
  echo "❌ Some services unhealthy: $UNHEALTHY"
  $COMPOSE_CMD logs
  exit 1
fi

# Clean up
$COMPOSE_CMD down -v

echo "✅ All Containerized Development Environment criteria met"
```
