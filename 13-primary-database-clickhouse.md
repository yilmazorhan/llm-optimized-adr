# Issue: Primary Database Selection for Cloud-Native Analytics Platform

Cloud-native microservices require a high-performance, scalable database solution optimized for analytical workloads, time-series data, and real-time querying. Traditional OLTP databases cannot efficiently handle the volume of event data, audit logs, metrics, and analytical queries required by modern observability and business intelligence systems.

# Decision:

Use **ClickHouse** as the primary database for analytical workloads, event storage, audit logging, and time-series data. ClickHouse provides columnar storage optimized for OLAP queries with exceptional compression and query performance characteristics suitable for cloud-native microservices architecture.

**Required Components**:
- ClickHouse 24.x LTS as the primary analytical database
- Quarkus ClickHouse client integration for reactive connectivity
- Column-oriented schema design optimized for analytical queries
- MergeTree engine family for efficient data storage and querying
- Materialized views for real-time aggregations
- Integration with existing observability stack (OpenTelemetry, Prometheus, Grafana)

# Constraints:

**Technical Constraints**:
- ClickHouse **MUST** be deployed using version 24.x LTS for stability and long-term support
- Database schema design **MUST** follow ClickHouse best practices for columnar storage
- Query patterns **MUST** be optimized for analytical (OLAP) workloads, not transactional (OLTP)
- Primary keys **MUST** be designed for optimal sorting and data locality
- Data ingestion **MUST** use batch inserts for optimal performance (avoid single-row inserts)
- Reactive client connections **MUST** integrate with Quarkus Mutiny patterns
- **MUST** support GraalVM native compilation for Quarkus applications

**Performance Constraints**:
- Query response time **MUST** be <100ms for common analytical queries on datasets up to 1TB
- Data compression ratio **SHOULD** achieve 10:1 or better for typical event data
- Ingestion throughput **MUST** support 100,000+ events per second
- Memory usage **MUST** remain within container resource limits

**Operational Constraints**:
- **MUST** support Docker and Kubernetes deployment models
- **MUST** integrate with existing backup and disaster recovery procedures
- **MUST** provide metrics compatible with Prometheus monitoring
- **MUST NOT** replace PostgreSQL for transactional CRUD operations requiring ACID guarantees
- **MUST** include Tabix Web UI in all development environments for database management and query exploration

# Alternatives:

**PostgreSQL**: Row-oriented storage engine designed for OLTP workloads (https://www.postgresql.org/docs/current/intro-whatis.html). ClickHouse benchmark results show columnar engines outperform PostgreSQL by 100–1000× on analytical aggregations over billion-row datasets (https://clickhouse.com/benchmark/dbms). PostgreSQL lacks native columnar storage and requires full-row reads for column-subset queries. Rejected because the project's primary workload—time-series event aggregation and audit log analytics—is OLAP, not OLTP.

**TimescaleDB**: PostgreSQL extension adding time-series hypertables with optional columnar compression (https://docs.timescale.com/about/latest/). TimescaleDB columnar compression is limited to completed chunks and does not support real-time inserts into compressed segments (https://docs.timescale.com/use-timescale/latest/compression/). ClickHouse MergeTree applies compression at write time with no chunk lifecycle dependency. Rejected because project ingestion rates (100 k+ events/s) require continuous compressed writes.

# Rationale:

ClickHouse stores data in compressed columnar format and uses vectorized query execution, achieving 10–50× compression on typical log/event datasets (https://clickhouse.com/docs/en/introduction). Independent benchmarks on the ClickBench suite confirm sub-second aggregation over billion-row tables where row-oriented alternatives exceed 10 s (https://benchmark.clickhouse.com/). The server ships as a single statically linked binary with built-in Docker and Kubernetes Helm chart support (https://clickhouse.com/docs/en/install), removing the need for external extension management. ClickHouse implements a large SQL dialect subset (https://clickhouse.com/docs/en/sql-reference) allowing standard query tooling while PostgreSQL remains the transactional (OLTP) store for ACID-dependent CRUD operations.

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

## 4. ClickHouse Client Configuration

```java
package com.copilot.quarkus.infrastructure.clickhouse;

import com.clickhouse.client.ClickHouseClient;
import com.clickhouse.client.ClickHouseNode;
import com.clickhouse.client.ClickHouseProtocol;
import com.clickhouse.client.ClickHouseRequest;
import com.clickhouse.client.ClickHouseResponse;
import io.quarkus.runtime.annotations.RegisterForReflection;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.CompletableFuture;

/**
 * ClickHouse client configuration and connection management.
 * 
 * <p>Provides reactive access to ClickHouse database operations
 * integrated with Quarkus Mutiny patterns.
 */
@ApplicationScoped
@RegisterForReflection
public class ClickHouseClientConfig {
    
    private static final Logger logger = LoggerFactory.getLogger(ClickHouseClientConfig.class);
    
    @ConfigProperty(name = "clickhouse.host", defaultValue = "localhost")
    String host;
    
    @ConfigProperty(name = "clickhouse.port", defaultValue = "8123")
    int port;
    
    @ConfigProperty(name = "clickhouse.database", defaultValue = "demo")
    String database;
    
    @ConfigProperty(name = "clickhouse.username", defaultValue = "demo")
    String username;
    
    @ConfigProperty(name = "clickhouse.password", defaultValue = "demo")
    String password;
    
    private ClickHouseNode server;
    
    @jakarta.annotation.PostConstruct
    void init() {
        server = ClickHouseNode.builder()
            .host(host)
            .port(ClickHouseProtocol.HTTP, port)
            .database(database)
            .credentials(username, password)
            .build();
        
        logger.info("ClickHouse client configured for {}:{}/{}", host, port, database);
    }
    
    /**
     * Execute a query and return results reactively.
     */
    public Uni<ClickHouseResponse> query(String sql) {
        return Uni.createFrom().completionStage(() -> {
            try (ClickHouseClient client = ClickHouseClient.newInstance(server.getProtocol())) {
                ClickHouseRequest<?> request = client.read(server)
                    .query(sql);
                return request.execute();
            }
        });
    }
    
    /**
     * Execute an insert operation reactively.
     */
    public Uni<ClickHouseResponse> insert(String tableName, String format, String data) {
        return Uni.createFrom().completionStage(() -> {
            try (ClickHouseClient client = ClickHouseClient.newInstance(server.getProtocol())) {
                String sql = String.format("INSERT INTO %s FORMAT %s", tableName, format);
                ClickHouseRequest<?> request = client.write(server)
                    .query(sql)
                    .data(data);
                return request.execute();
            }
        });
    }
    
    public ClickHouseNode getServer() {
        return server;
    }
}
```

## 5. Schema Design Best Practices

### Audit Event Table Example

```sql
-- Audit events table optimized for time-series analytical queries
CREATE TABLE IF NOT EXISTS audit_events (
    -- Event identification
    event_id UUID DEFAULT generateUUIDv4(),
    timestamp DateTime64(3) DEFAULT now64(3),
    
    -- Service context
    service_name LowCardinality(String),
    service_version LowCardinality(String),
    host_name LowCardinality(String),
    
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
    actor_ip IPv4,
    
    -- Resource information
    resource_type LowCardinality(String),
    resource_id String,
    
    -- Event data (JSON for flexibility)
    event_data String CODEC(ZSTD(3)),
    
    -- Diff data for updates
    diff_data String CODEC(ZSTD(3))
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (service_name, event_name, timestamp)
TTL timestamp + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;

-- Create materialized view for real-time event aggregations
CREATE MATERIALIZED VIEW IF NOT EXISTS audit_events_hourly_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (service_name, event_name, outcome, hour)
AS SELECT
    service_name,
    event_name,
    outcome,
    toStartOfHour(timestamp) AS hour,
    count() AS event_count
FROM audit_events
GROUP BY service_name, event_name, outcome, hour;
```

### Application Metrics Table Example

```sql
-- Application metrics for observability
CREATE TABLE IF NOT EXISTS application_metrics (
    timestamp DateTime64(3) DEFAULT now64(3),
    
    -- Metric identification
    metric_name LowCardinality(String),
    metric_type LowCardinality(Enum8('COUNTER' = 1, 'GAUGE' = 2, 'HISTOGRAM' = 3)),
    
    -- Dimensions
    service_name LowCardinality(String),
    endpoint LowCardinality(String),
    status_code UInt16,
    
    -- Values
    value Float64,
    count UInt64,
    sum Float64,
    
    -- Labels (for additional dimensions)
    labels Map(String, String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (service_name, metric_name, timestamp)
TTL timestamp + INTERVAL 30 DAY
SETTINGS index_granularity = 8192;
```

## 6. Repository Adapter Implementation

```java
package com.copilot.quarkus.infrastructure.clickhouse.adapter;

import com.copilot.quarkus.domain.audit.model.AuditEvent;
import com.copilot.quarkus.domain.audit.port.AuditEventRepositoryPort;
import com.copilot.quarkus.infrastructure.clickhouse.ClickHouseClientConfig;
import io.quarkus.runtime.annotations.RegisterForReflection;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;
import java.util.List;

/**
 * ClickHouse adapter for audit event persistence.
 * 
 * <p>Implements the audit event repository port using ClickHouse
 * for high-performance analytical storage and querying.
 */
@ApplicationScoped
@RegisterForReflection
public class ClickHouseAuditEventAdapter implements AuditEventRepositoryPort {
    
    private static final Logger logger = LoggerFactory.getLogger(ClickHouseAuditEventAdapter.class);
    
    @Inject
    ClickHouseClientConfig clickHouseClient;
    
    @Override
    public Uni<Void> save(AuditEvent event) {
        String insertSql = buildInsertStatement(event);
        
        return clickHouseClient.query(insertSql)
            .onItem().invoke(response -> 
                logger.debug("Audit event saved: {}", event.getEventId()))
            .onFailure().invoke(ex -> 
                logger.error("Failed to save audit event: {}", event.getEventId(), ex))
            .replaceWithVoid();
    }
    
    @Override
    public Uni<Void> saveBatch(List<AuditEvent> events) {
        if (events.isEmpty()) {
            return Uni.createFrom().voidItem();
        }
        
        StringBuilder batchData = new StringBuilder();
        for (AuditEvent event : events) {
            batchData.append(formatEventAsJSON(event)).append("\n");
        }
        
        return clickHouseClient.insert("audit_events", "JSONEachRow", batchData.toString())
            .onItem().invoke(response -> 
                logger.debug("Batch of {} audit events saved", events.size()))
            .replaceWithVoid();
    }
    
    @Override
    public Multi<AuditEvent> findByServiceAndTimeRange(
            String serviceName, 
            Instant startTime, 
            Instant endTime) {
        
        // Use parameterized queries to prevent SQL injection
        String sql = """
            SELECT *
            FROM audit_events
            WHERE service_name = {serviceName:String}
              AND timestamp >= toDateTime64({startTime:String}, 3)
              AND timestamp <= toDateTime64({endTime:String}, 3)
            ORDER BY timestamp DESC
            LIMIT 1000
            """;
        
        return clickHouseClient.query(sql,
                Map.of("serviceName", serviceName,
                       "startTime", startTime.toString(),
                       "endTime", endTime.toString()))
            .onItem().transformToMulti(response -> 
                Multi.createFrom().iterable(parseAuditEvents(response)));
    }
    
    @Override
    public Uni<Long> countByEventName(String eventName, Instant since) {
        // Use parameterized queries to prevent SQL injection
        String sql = """
            SELECT count() AS cnt
            FROM audit_events
            WHERE event_name = {eventName:String}
              AND timestamp >= toDateTime64({since:String}, 3)
            """;
        
        return clickHouseClient.query(sql,
                Map.of("eventName", eventName,
                       "since", since.toString()))
            .onItem().transform(response -> extractCount(response));
    }
    
    private String buildInsertStatement(AuditEvent event) {
        // Use parameterized queries to prevent SQL injection
        String sql = """
            INSERT INTO audit_events (
                event_id, timestamp, service_name, service_version,
                event_name, event_category, outcome,
                correlation_id, trace_id, actor_type, actor_id,
                resource_type, resource_id, event_data
            ) VALUES (
                {eventId:String}, now64(3), {serviceName:String}, {serviceVersion:String},
                {eventName:String}, {eventCategory:String}, {outcome:String},
                {correlationId:String}, {traceId:String}, {actorType:String}, {actorId:String},
                {resourceType:String}, {resourceId:String}, {eventData:String}
            )
            """;
        return clickHouseClient.buildParameterized(sql,
            Map.ofEntries(
                Map.entry("eventId", event.getEventId()),
                Map.entry("serviceName", event.getServiceName()),
                Map.entry("serviceVersion", event.getServiceVersion()),
                Map.entry("eventName", event.getEventName()),
                Map.entry("eventCategory", event.getEventCategory()),
                Map.entry("outcome", event.getOutcome()),
                Map.entry("correlationId", event.getCorrelationId()),
                Map.entry("traceId", event.getTraceId()),
                Map.entry("actorType", event.getActorType()),
                Map.entry("actorId", event.getActorId()),
                Map.entry("resourceType", event.getResourceType()),
                Map.entry("resourceId", event.getResourceId()),
                Map.entry("eventData", event.getEventData() != null ? event.getEventData() : "")
            ));
    }
    
    private String formatEventAsJSON(AuditEvent event) {
        // Use a JSON library (e.g., Jackson ObjectMapper) for safe serialization
        // in production code. Inline formatting shown for brevity.
        return String.format("""
            {"event_id":"%s","service_name":"%s","event_name":"%s","outcome":"%s"}
            """, escapeJsonValue(event.getEventId()), escapeJsonValue(event.getServiceName()), 
                 escapeJsonValue(event.getEventName()), escapeJsonValue(event.getOutcome()));
    }
    
    private String escapeJsonValue(String value) {
        if (value == null) return "";
        return value.replace("\\", "\\\\")
                    .replace("\"", "\\\"")
                    .replace("\n", "\\n")
                    .replace("\r", "\\r")
                    .replace("\t", "\\t");
    }
    
    // Additional helper methods...
}
```

## 7. Validation Commands

```bash
# Verify ClickHouse container is running
docker compose ps clickhouse

# Connect to ClickHouse CLI
docker compose exec clickhouse clickhouse-client

# Test database connectivity
docker compose exec clickhouse clickhouse-client --query "SELECT version()"

# Verify database creation
docker compose exec clickhouse clickhouse-client --query "SHOW DATABASES"

# Check table schemas
docker compose exec clickhouse clickhouse-client --query "SHOW TABLES FROM demo"

# Test query performance
docker compose exec clickhouse clickhouse-client --query "
  SELECT 
    service_name, 
    event_name, 
    count() as cnt 
  FROM demo.audit_events 
  GROUP BY service_name, event_name 
  ORDER BY cnt DESC 
  LIMIT 10
"

# Verify compression ratio
docker compose exec clickhouse clickhouse-client --query "
  SELECT 
    table,
    formatReadableSize(sum(bytes)) AS size,
    formatReadableSize(sum(bytes_on_disk)) AS compressed,
    round(sum(bytes) / sum(bytes_on_disk), 2) AS ratio
  FROM system.parts
  WHERE database = 'demo'
  GROUP BY table
"

# Health check endpoint
curl -s "http://localhost:8123/ping"

# Application integration test
./mvnw test -Dtest=*ClickHouse*IntegrationTest
```

# Additional Recommendations:

- It is RECOMMENDED to use batch inserts (1 000–10 000 rows) with ZSTD codec for optimal ingestion throughput (https://clickhouse.com/docs/en/sql-reference/statements/insert-into)
- It is RECOMMENDED to create materialized views for frequently accessed aggregations and use Dictionaries for dimension lookups (https://clickhouse.com/docs/en/sql-reference/dictionaries)
- It is RECOMMENDED to implement incremental backups using `clickhouse-backup` (https://github.com/Altinity/clickhouse-backup) and enable `system.metrics` export to Prometheus (https://clickhouse.com/docs/en/operations/monitoring)
- It is RECOMMENDED to configure ReplicatedMergeTree engines with ClickHouse Keeper for production high-availability (https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication)
- PostgreSQL MUST continue to serve CRUD operations requiring ACID transactions (see ADR-08, ADR-10)

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| ClickHouse Driver | `grep clickhouse pom.xml` | JDBC driver present |
| ClickHouse Config | `grep clickhouse application.properties` | Connection configured |
| Docker Compose | `grep clickhouse docker-compose.yml` | ClickHouse service defined |
| Health Check | Container health | ClickHouse container healthy |
| Connection Test | `SELECT 1` query | Query executes successfully |
| Schema Migration | Migration files | ClickHouse DDL exists |
| Repository Adapter | Code inspection | ClickHouse repository impl |
| Testcontainers | Test code | ClickHouseContainer used |
| Integration Tests Pass | `./mvnw verify` | Exit code 0 |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Primary Database - ClickHouse..."

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
if grep -rq "clickhouse\|8123\|9000" */src/main/resources/application*.properties 2>/dev/null; then
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

# Check for ClickHouse repository
if grep -rq "ClickHouse\|clickhouse" */src/main/java/**/infrastructure 2>/dev/null; then
  echo "✅ ClickHouse repository adapter found"
else
  echo "❌ No ClickHouse repository adapter"
  FAIL=1
fi

# Check for ClickHouse migrations
if find . -name "*.sql" -path "*migration*" -exec grep -l "ENGINE.*MergeTree\|CREATE TABLE.*clickhouse" {} \; 2>/dev/null | grep -q .; then
  echo "✅ ClickHouse schema migrations found"
else
  echo "❌ No ClickHouse migrations"
  FAIL=1
fi

# Check for ClickHouse Testcontainers
if grep -rq "ClickHouseContainer\|clickhouse/clickhouse-server" */src/test/java 2>/dev/null; then
  echo "✅ ClickHouse Testcontainers integration found"
else
  echo "❌ ClickHouse Testcontainers integration not found"
  FAIL=1
fi

# Test ClickHouse connection (if running)
if docker ps 2>/dev/null | grep -q clickhouse; then
  echo "Testing ClickHouse connection..."
  docker exec $(docker ps -q -f name=clickhouse | head -1) clickhouse-client --query "SELECT 1" 2>/dev/null && echo "✅ ClickHouse connection successful" || { echo "❌ ClickHouse query failed"; FAIL=1; }
else
  echo "ℹ️  ClickHouse container not running (skipping live connection test)"
fi

# Build verification
./mvnw clean compile -q && echo "✅ Build successful" || { echo "❌ Build failed"; FAIL=1; }

if [ $FAIL -ne 0 ]; then
  echo "❌ Primary Database - ClickHouse validation FAILED"
  exit 1
fi
echo "✅ All Primary Database - ClickHouse criteria met"
```
