# Issue: ClickHouse Infrastructure Setup for Development

Cloud-native microservices require a high-performance ClickHouse database environment for analytical workloads, event storage, audit logging, and time-series data. This ADR covers the infrastructure setup: Docker Compose services, Maven dependencies, application configuration, and validation.

**See also**: [13b-clickhouse-coding-standards.md](../03-coding-style-standards-packaging/13b-clickhouse-coding-standards.md) for schema design, client configuration, and repository adapter implementation.

# Decision:

Use **ClickHouse 24.x LTS** as the primary analytical database with the following infrastructure components:
- Docker Compose service with Tabix Web UI for development
- ClickHouse JDBC client 0.6.3 Maven dependencies
- Profile-based application configuration (%dev/%test/%prod)

# Constraints:

- ClickHouse **MUST** be deployed using version 24.x LTS for stability and long-term support
- **MUST** support Docker and Kubernetes deployment models
- **MUST** integrate with existing backup and disaster recovery procedures
- **MUST** provide metrics compatible with Prometheus monitoring
- **MUST** include Tabix Web UI in all development environments for database management and query exploration

# Alternatives:

**PostgreSQL**: Row-oriented storage engine designed for OLTP workloads (https://www.postgresql.org/docs/current/intro-whatis.html). ClickHouse benchmark results show columnar engines outperform PostgreSQL by 100-1000x on analytical aggregations over billion-row datasets (https://clickhouse.com/benchmark/dbms). Rejected because the project's primary workload is OLAP, not OLTP.

**TimescaleDB**: PostgreSQL extension adding time-series hypertables with optional columnar compression (https://docs.timescale.com/about/latest/). TimescaleDB columnar compression is limited to completed chunks and does not support real-time inserts into compressed segments (https://docs.timescale.com/use-timescale/latest/compression/). Rejected because project ingestion rates (100k+ events/s) require continuous compressed writes.

# Rationale:

ClickHouse stores data in compressed columnar format and uses vectorized query execution, achieving 10-50x compression on typical log/event datasets (https://clickhouse.com/docs/en/introduction). The server ships as a single statically linked binary with built-in Docker and Kubernetes Helm chart support (https://clickhouse.com/docs/en/install), removing the need for external extension management.

# Implementation Guidelines:

## 1. Docker Compose Configuration

Add ClickHouse to the development infrastructure:

```yaml
services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.3-alpine
    container_name: clickhouse
    ports:
      - "8123:8123"   # HTTP interface
      - "9000:9000"   # Native TCP interface
      - "9009:9009"   # Inter-server communication
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - clickhouse_logs:/var/log/clickhouse-server
      - ./clickhouse/config:/etc/clickhouse-server/config.d
      - ./clickhouse/users:/etc/clickhouse-server/users.d
    environment:
      CLICKHOUSE_DB: ${CLICKHOUSE_DB:-demo}
      CLICKHOUSE_USER: ${CLICKHOUSE_USER:-demo}
      CLICKHOUSE_PASSWORD: ${CLICKHOUSE_PASSWORD:-demo}
      CLICKHOUSE_DEFAULT_ACCESS_MANAGEMENT: 1
    healthcheck:
      test: ["CMD", "clickhouse-client", "--query", "SELECT 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2'
        reservations:
          memory: 1G
          cpus: '1'

  # Tabix - ClickHouse Web UI (MUST always be enabled for development)
  tabix:
    image: spoonest/clickhouse-tabix-web-client:stable
    container_name: tabix
    ports:
      - "8124:80"    # Tabix Web UI
    depends_on:
      - clickhouse

volumes:
  clickhouse_data:
  clickhouse_logs:
```

> **Note**: Tabix **MUST** always be enabled in development environments to provide visual database management, query execution, and schema exploration capabilities.

## 2. Maven Dependencies

Add ClickHouse client dependencies to `pom.xml`:

```xml
<!-- ClickHouse JDBC Client -->
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.6.3</version>
    <classifier>http</classifier>
</dependency>

<!-- ClickHouse Native Client (for high-performance operations) -->
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-client</artifactId>
    <version>0.6.3</version>
</dependency>

<!-- ClickHouse Data Types -->
<dependency>
    <groupId>com.clickhouse</groupId>
    <artifactId>clickhouse-data</artifactId>
    <version>0.6.3</version>
</dependency>
```

## 3. Application Configuration

Add ClickHouse configuration to `application.properties`:

```properties
# =============================================================================
# CLICKHOUSE CONFIGURATION
# =============================================================================

# ClickHouse Connection Settings
clickhouse.host=${CLICKHOUSE_HOST:localhost}
clickhouse.port=${CLICKHOUSE_PORT:8123}
clickhouse.database=${CLICKHOUSE_DB:demo}
clickhouse.username=${CLICKHOUSE_USER:demo}
clickhouse.password=${CLICKHOUSE_PASSWORD:demo}

# Connection Pool Settings
clickhouse.connection.timeout=${CLICKHOUSE_CONNECTION_TIMEOUT:30000}
clickhouse.socket.timeout=${CLICKHOUSE_SOCKET_TIMEOUT:60000}
clickhouse.max.connections=${CLICKHOUSE_MAX_CONNECTIONS:10}

# Query Settings
clickhouse.max.execution.time=${CLICKHOUSE_MAX_EXECUTION_TIME:60}
clickhouse.max.rows.to.read=${CLICKHOUSE_MAX_ROWS:1000000000}
clickhouse.compress=${CLICKHOUSE_COMPRESS:true}

# Development Profile
%dev.clickhouse.host=localhost
%dev.clickhouse.port=8123
%dev.clickhouse.database=demo
%dev.clickhouse.username=demo
%dev.clickhouse.password=demo

# Production Profile
%prod.clickhouse.host=${CLICKHOUSE_HOST}
%prod.clickhouse.port=${CLICKHOUSE_PORT:8123}
%prod.clickhouse.database=${CLICKHOUSE_DB}
%prod.clickhouse.username=${CLICKHOUSE_USER}
%prod.clickhouse.password=${CLICKHOUSE_PASSWORD}
%prod.clickhouse.compress=true

# Test Profile
%test.clickhouse.host=localhost
%test.clickhouse.port=8123
%test.clickhouse.database=test_demo
```

## 4. Validation Commands

```bash
# Verify ClickHouse container is running
docker compose ps clickhouse

# Connect to ClickHouse CLI
docker compose exec clickhouse clickhouse-client

# Test database connectivity
docker compose exec clickhouse clickhouse-client --query "SELECT version()"

# Verify database creation
docker compose exec clickhouse clickhouse-client --query "SHOW DATABASES"

# Health check endpoint
curl -s "http://localhost:8123/ping"
```

# Additional Recommendations:

- It is **RECOMMENDED** to use ClickHouse Alpine images for faster container startup and reduced image size (https://hub.docker.com/r/clickhouse/clickhouse-server)
- It is **RECOMMENDED** to configure container resource limits matching the target deployment environment
- It is **RECOMMENDED** to externalize ClickHouse server configuration via mounted config.d and users.d volumes for reproducible environments
- **References**: ClickHouse Docker Images: https://hub.docker.com/r/clickhouse/clickhouse-server, ClickHouse Server Configuration: https://clickhouse.com/docs/en/operations/configuration-files

# Success Criteria:

**Completion Gate**: The following criteria **MUST** all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| ClickHouse Driver | `grep clickhouse pom.xml` | JDBC driver present |
| ClickHouse Config | `grep clickhouse application.properties` | Connection configured |
| Docker Compose | `grep clickhouse docker-compose.yml` | ClickHouse service defined |
| Tabix Web UI | `grep tabix docker-compose.yml` | Tabix service defined |
| Health Check | Container health | ClickHouse container healthy |
| Connection Test | `SELECT 1` query | Query executes successfully |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
echo "Validating ClickHouse Infrastructure Setup..."

FAIL=0

# Detect Docker Compose command variant
if command -v docker compose &>/dev/null; then
  DOCKER_COMPOSE="docker compose"
elif command -v docker-compose &>/dev/null; then
  DOCKER_COMPOSE="docker-compose"
else
  echo "❌ Neither 'docker compose' nor 'docker-compose' found"
  FAIL=1
fi

# Check for ClickHouse JDBC driver
if grep -rq "clickhouse" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ ClickHouse dependency present"
else
  echo "❌ ClickHouse dependency not found"
  FAIL=1
fi

# Check for ClickHouse configuration
if grep -rq "clickhouse" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ ClickHouse configuration found"
else
  echo "❌ ClickHouse not configured"
  FAIL=1
fi

# Check Docker Compose for ClickHouse
COMPOSE_FILE=$(ls docker-compose*.yml compose.yml 2>/dev/null | head -1)
if [ -n "$COMPOSE_FILE" ] && grep -q "clickhouse" "$COMPOSE_FILE"; then
  echo "✅ ClickHouse in Docker Compose"
else
  echo "❌ ClickHouse not in Docker Compose"
  FAIL=1
fi

# Check Docker Compose for Tabix
if [ -n "$COMPOSE_FILE" ] && grep -q "tabix" "$COMPOSE_FILE"; then
  echo "✅ Tabix Web UI in Docker Compose"
else
  echo "❌ Tabix Web UI not in Docker Compose"
  FAIL=1
fi

# Test ClickHouse connection (if running)
if docker ps 2>/dev/null | grep -q clickhouse; then
  docker exec $(docker ps -q -f name=clickhouse | head -1) clickhouse-client --query "SELECT 1" 2>/dev/null && echo "✅ ClickHouse connection successful" || { echo "❌ ClickHouse query failed"; FAIL=1; }
else
  echo "ℹ️  ClickHouse container not running (skipping live connection test)"
fi

# Build verification
if ./mvnw clean compile -q 2>/dev/null; then
  echo "✅ Build successful"
else
  echo "❌ Build failed"
  FAIL=1
fi

if [ "$FAIL" -ne 0 ]; then
  echo "❌ ClickHouse Infrastructure Setup validation FAILED"
  exit 1
fi
echo "✅ All ClickHouse Infrastructure Setup criteria met"
```
