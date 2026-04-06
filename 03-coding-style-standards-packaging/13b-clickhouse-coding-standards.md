# Issue: ClickHouse Schema Design, Client Configuration, and Repository Implementation

ClickHouse requires specific coding patterns for schema design, client integration, and repository adapters that differ from traditional RDBMS approaches. This ADR defines the coding standards for working with ClickHouse in the application layer.

**See also**: [13a-clickhouse-infrastructure-setup.md](../01-development-environment-setup/13a-clickhouse-infrastructure-setup.md) for Docker Compose, Maven dependencies, and application configuration.

# Decision:

All ClickHouse application code **MUST** follow columnar-optimized patterns: MergeTree engine family for storage, batch inserts for ingestion, reactive Mutiny integration for queries, and hexagonal architecture adapters for repository implementation.

# Constraints:

- Database schema design **MUST** follow ClickHouse best practices for columnar storage
- Query patterns **MUST** be optimized for analytical (OLAP) workloads, not transactional (OLTP)
- Primary keys **MUST** be designed for optimal sorting and data locality
- Data ingestion **MUST** use batch inserts for optimal performance (avoid single-row inserts)
- Reactive client connections **MUST** integrate with Quarkus Mutiny patterns
- **MUST** support GraalVM native compilation for Quarkus applications
- Query response time **MUST** be <100ms for common analytical queries on datasets up to 1TB
- Data compression ratio **SHOULD** achieve 10:1 or better for typical event data
- Ingestion throughput **MUST** support 100,000+ events per second
- **MUST NOT** replace PostgreSQL for transactional CRUD operations requiring ACID guarantees

# Alternatives:

**Raw JDBC with Manual Connection Management**: Direct `java.sql.DriverManager` usage without reactive wrappers (https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/DriverManager.html). Blocking JDBC calls on Quarkus reactive threads violate the Vert.x event loop contract, causing `BlockingOperationException` at runtime (https://quarkus.io/guides/quarkus-reactive-architecture). Rejected because all database access **MUST** integrate with Mutiny reactive patterns per ADR-08.

**Spring Data JPA Repository Abstraction**: Spring Data JPA generates query implementations from interface method names (https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html). Spring Data JPA targets row-oriented JPA entities and does not support ClickHouse MergeTree engine configuration, `PARTITION BY`, `ORDER BY` tuple keys, or `CODEC` column compression directives. Rejected because ClickHouse-specific DDL and query patterns cannot be expressed through JPA annotations.

# Rationale:

Independent benchmarks on the ClickBench suite confirm sub-second aggregation over billion-row tables where row-oriented alternatives exceed 10s (https://benchmark.clickhouse.com/). ClickHouse implements a large SQL dialect subset (https://clickhouse.com/docs/en/sql-reference) allowing standard query tooling. The ClickHouse JDBC driver 0.6.3 supports `CompletableFuture`-based async execution (https://github.com/ClickHouse/clickhouse-java), which maps directly to Mutiny `Uni.createFrom().completionStage()` for non-blocking integration. PostgreSQL remains the transactional (OLTP) store for ACID-dependent CRUD operations.

# Implementation Guidelines:

## 1. ClickHouse Client Configuration

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

## 2. Schema Design Best Practices

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

## 3. Naming Conventions

- Client configuration class: `ClickHouseClientConfig` in `infrastructure.clickhouse` package
- Repository adapters: `ClickHouse{Entity}Adapter` in `infrastructure.clickhouse.adapter` package
- Repository ports: `{Entity}RepositoryPort` in `domain.{context}.port` package
- SQL migration files: `V{version}__clickhouse_{description}.sql`
- Table names: `snake_case` (e.g., `audit_events`, `application_metrics`)
- Column names: `snake_case` (e.g., `event_name`, `trace_id`)
- Materialized views: `{table_name}_{aggregation}_mv` (e.g., `audit_events_hourly_mv`)

## 4. Repository Adapter Implementation

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

# Additional Recommendations:

- It is **RECOMMENDED** to use batch inserts (1,000-10,000 rows) with ZSTD codec for optimal ingestion throughput (https://clickhouse.com/docs/en/sql-reference/statements/insert-into)
- It is **RECOMMENDED** to create materialized views for frequently accessed aggregations and use Dictionaries for dimension lookups (https://clickhouse.com/docs/en/sql-reference/dictionaries)
- It is **RECOMMENDED** to implement incremental backups using `clickhouse-backup` (https://github.com/Altinity/clickhouse-backup) and enable `system.metrics` export to Prometheus (https://clickhouse.com/docs/en/operations/monitoring)
- It is **RECOMMENDED** to configure ReplicatedMergeTree engines with ClickHouse Keeper for production high-availability (https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication)
- PostgreSQL **MUST** continue to serve CRUD operations requiring ACID transactions (see ADR-08, ADR-10)
- **References**: ClickHouse SQL Reference: https://clickhouse.com/docs/en/sql-reference, ClickHouse Java Client: https://github.com/ClickHouse/clickhouse-java, ClickHouse MergeTree Engines: https://clickhouse.com/docs/en/engines/table-engines/mergetree-family

# Success Criteria:

**Completion Gate**: The following criteria **MUST** all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Schema Migration | `find . -name "*.sql" -path "*migration*"` | ClickHouse DDL files exist |
| Repository Adapter | `grep -r "ClickHouseAuditEventAdapter" */src/main/java` | Adapter class implemented |
| Repository Port | `grep -r "AuditEventRepositoryPort" */src/main/java` | Port interface defined |
| Reactive Integration | `grep -r "Uni\|Multi" */src/main/java/**/clickhouse` | Mutiny types used in adapters |
| Batch Insert Support | `grep -r "saveBatch\|JSONEachRow" */src/main/java` | Batch insert method implemented |
| Parameterized Queries | `grep -r "parameterized\|{.*:String}" */src/main/java` | SQL injection prevention |
| Integration Tests Pass | `./mvnw verify` | Exit code 0 |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
echo "Validating ClickHouse Coding Standards..."

FAIL=0

# Detect test and source paths
SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
TEST_PATHS=$(find . -path "*/src/test/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)

# Check for ClickHouse schema migrations
if find . -name "*.sql" -path "*migration*" -exec grep -l "ENGINE.*MergeTree\|CREATE TABLE.*audit_events" {} \; 2>/dev/null | grep -q .; then
  echo "✅ ClickHouse schema migrations found"
else
  echo "❌ No ClickHouse schema migrations"
  FAIL=1
fi

# Check for repository adapter
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "ClickHouse.*Adapter\|implements.*RepositoryPort" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ ClickHouse repository adapter found"
else
  echo "❌ No ClickHouse repository adapter"
  FAIL=1
fi

# Check for repository port interface
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "interface.*RepositoryPort" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Repository port interface found"
else
  echo "❌ No repository port interface"
  FAIL=1
fi

# Check for Mutiny reactive types in ClickHouse code
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "Uni\|Multi" "$sp"/**/clickhouse* 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Reactive Mutiny integration in ClickHouse code"
else
  echo "❌ No Mutiny types in ClickHouse code"
  FAIL=1
fi

# Check for batch insert support
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "saveBatch\|JSONEachRow" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Batch insert support implemented"
else
  echo "❌ No batch insert support"
  FAIL=1
fi

# Check for parameterized queries (SQL injection prevention)
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "parameterized\|{.*:String}" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Parameterized queries used"
else
  echo "❌ No parameterized queries found"
  FAIL=1
fi

# Build verification
if ./mvnw clean compile -q 2>/dev/null; then
  echo "✅ Build successful"
else
  echo "❌ Build failed"
  FAIL=1
fi

if [ "$FAIL" -ne 0 ]; then
  echo "❌ ClickHouse Coding Standards validation FAILED"
  exit 1
fi
echo "✅ All ClickHouse Coding Standards criteria met"
```
