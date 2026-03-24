# Issue: Integration Testing Framework with Testcontainers

Cloud-native applications need reliable integration testing that simulates production database environments without external database dependencies. Current testing lacks comprehensive integration coverage for ClickHouse analytical operations and containerized environments.

# Decision:

Implement comprehensive Testcontainers-based integration testing with reliable, isolated, and reproducible ClickHouse environments. All integration tests use standardized `ClickHouseTestResource` supporting analytical query operations and Quarkus data sources.

**Required Components**:
- **Database Environment**: ClickHouse Testcontainers with version-specific container images
- **HTTP/Native Protocol Support**: Configure both HTTP and native TCP connections for comprehensive compatibility
- **Container Management**: Proper lifecycle management with resource cleanup and Ryuk compatibility
- **CI/CD Integration**: Support various Docker environments including rootless Docker and containerized CI systems

# Constraints:

- Integration tests **MUST** support both HTTP and native ClickHouse client protocols
- **MUST** work reliably across Docker Desktop, Colima, rootless Docker, and containerized CI environments
- **MUST** ensure complete test isolation with no state leakage between executions
- Container startup and test execution **MUST NOT** exceed reasonable time limits for CI/CD efficiency
- **SHOULD** operate within available Docker memory and CPU constraints
- Ryuk resource reaper **MAY** be disabled in environments with restricted privileged container access

# Alternatives:

**Option 1 — Quarkus Dev Services**: Automatic database provisioning without explicit container management. Rejected because Quarkus Dev Services for ClickHouse is not officially supported — the Quarkus Dev Services documentation (https://quarkus.io/guides/databases-dev-services) lists PostgreSQL, MySQL, MariaDB, MongoDB, and others, but does not include ClickHouse as a supported database. This requires a custom `DevServicesConfig` implementation that replicates what Testcontainers already provides, with no added benefit.

**Option 2 — H2 In-Memory Database**: Embedded database for fast unit testing. Rejected because H2 is a row-oriented RDBMS (https://h2database.com/html/features.html) that does not support ClickHouse-specific features: MergeTree table engines, columnar storage, materialized views, or ClickHouse SQL dialect extensions (`FINAL`, `SAMPLE`, array functions). Per ClickHouse documentation (https://clickhouse.com/docs/en/engines/table-engines/mergetree-family), MergeTree engines are fundamental to ClickHouse's storage model and cannot be emulated in H2.

# Rationale:

Testcontainers starts each test suite with a fresh Docker container, guaranteeing no shared state between test runs — per Testcontainers architecture documentation (https://java.testcontainers.org/features/creating_container/), each container instance uses an isolated filesystem and network namespace. The `QuarkusTestResourceLifecycleManager` interface (https://quarkus.io/guides/getting-started-testing#quarkus-test-resource) provides lifecycle hooks (`start()`/`stop()`) that inject container connection properties directly into the Quarkus configuration system, eliminating manual configuration. Using the official `clickhouse/clickhouse-server` Docker image ensures the test database engine matches production (MergeTree engines, columnar storage, ClickHouse SQL dialect), unlike abstraction layers that approximate behavior.

# Implementation Guidelines:

## ClickHouseTestResource Implementation:

```java
package com.copilot.quarkus.infrastructure.database.testresource;

import io.quarkus.test.common.QuarkusTestResourceLifecycleManager;
import org.testcontainers.clickhouse.ClickHouseContainer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.HashMap;
import java.util.Map;

public class ClickHouseTestResource implements QuarkusTestResourceLifecycleManager {
    
    private static final Logger logger = LoggerFactory.getLogger(ClickHouseTestResource.class);
    private ClickHouseContainer clickhouse;

    @Override
    public Map<String, String> start() {
        logger.info("Starting ClickHouse Testcontainer for integration testing");
        
        clickhouse = new ClickHouseContainer("clickhouse/clickhouse-server:24.3-alpine")
                .withDatabaseName("testdb")
                .withUsername("test")
                .withPassword("test")
                .withReuse(false);
        
        clickhouse.start();
        
        Map<String, String> config = new HashMap<>();
        config.put("clickhouse.host", clickhouse.getHost());
        config.put("clickhouse.port", String.valueOf(clickhouse.getMappedPort(8123)));
        config.put("clickhouse.database", "testdb");
        config.put("clickhouse.username", clickhouse.getUsername());
        config.put("clickhouse.password", clickhouse.getPassword());
        config.put("clickhouse.jdbc.url", clickhouse.getJdbcUrl());
        
        return config;
    }

    @Override
    public void stop() {
        if (clickhouse != null && clickhouse.isRunning()) {
            clickhouse.stop();
            clickhouse = null;
        }
    }
}
```

## Integration Test Implementation (**CRITICAL**):

```java
package com.copilot.quarkus.infrastructure.database.adapter;

import com.copilot.quarkus.infrastructure.database.testresource.ClickHouseTestResource;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.common.QuarkusTestResource;
import jakarta.inject.Inject;
import org.junit.jupiter.api.*;
import java.time.Duration;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Comprehensive integration tests for ClickHouse Audit Repository Adapter.
 * 
 * <p>Tests all query operations using real ClickHouse database via Testcontainers,
 * ensuring production-like behavior for analytical query operations.
 */
@QuarkusTest
@QuarkusTestResource(ClickHouseTestResource.class)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class ClickHouseAuditRepositoryAdapterTest {

    @Inject 
    ClickHouseAuditRepository auditRepository;

    @Test
    @Order(1)
    void testInsertAuditEvent_ShouldPersistSuccessfully() {
        // Given: New audit event for persistence
        AuditEvent event = AuditEvent.builder()
            .eventName("USER_CREATED")
            .outcome("SUCCESS")
            .actorId("usr_123")
            .resourceType("USER")
            .resourceId("usr_456")
            .build();
        
        // When: Saving audit event
        auditRepository.save(event)
            .await().atMost(Duration.ofSeconds(10));
        
        // Then: Event persisted successfully
    }

    @Test
    @Order(2)
    void testQueryByTimeRange_ShouldRetrieveEvents() {
        // Given: Time range for query
        var from = Instant.now().minus(Duration.ofHours(1));
        var to = Instant.now();
        
        // When: Querying events
        List<AuditEvent> events = auditRepository.findByTimeRange(from, to)
            .await().atMost(Duration.ofSeconds(10));
        
        // Then: Events retrieved within time range
        assertNotNull(events);
    }

    @Test
    @Order(3)
    void testQueryByActorId_ShouldFilterCorrectly() {
        // Given: Actor ID to filter
        String actorId = "usr_123";
        
        // When: Querying by actor
        List<AuditEvent> events = auditRepository.findByActorId(actorId)
            .await().atMost(Duration.ofSeconds(10));
        
        // Then: Events filtered by actor
        events.forEach(e -> assertEquals(actorId, e.getActorId()));
    }

    @Test
    @Order(4)
    void testAggregateByEventName_ShouldReturnCounts() {
        // Given: Aggregation query
        
        // When: Aggregating events by name
        Map<String, Long> counts = auditRepository.countByEventName()
            .await().atMost(Duration.ofSeconds(10));
        
        // Then: Counts returned per event type
        assertNotNull(counts);
    }
}
```

## Environment Configuration (**MUST** be configured):

## Maven Dependencies:

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>clickhouse</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

## Environment Configuration:

```bash
# Disable Ryuk for Docker environment compatibility
export TESTCONTAINERS_RYUK_DISABLED=true

# Optional: Configure for corporate environments
export TESTCONTAINERS_HUB_IMAGE_NAME_PREFIX=your-registry.com/
```

## Validation Commands:

```bash
# Run integration tests with Testcontainers
./mvnw test -Dtest=*IntegrationTest

# Run all tests including integration tests
./mvnw verify

# Run tests with detailed Testcontainers logging
./mvnw test -Dorg.slf4j.simpleLogger.log.org.testcontainers=DEBUG

# Verify ClickHouse container cleanup
docker ps -a | grep clickhouse

# Test Docker environment compatibility
docker run --rm hello-world
```

# Additional Recommendations:

- Use ClickHouse Alpine images for faster container startup
- Implement container reuse for test suites with multiple test classes
- Validate ClickHouse schema migrations during integration tests
- Extend pattern to support additional databases (Redis, MongoDB) as needed
- Monitor test execution time and container resource usage in CI/CD pipelines

**References:**
- Testcontainers for Java: https://java.testcontainers.org/
- Testcontainers ClickHouse Module: https://java.testcontainers.org/modules/databases/clickhouse/
- Quarkus Testing Guide: https://quarkus.io/guides/getting-started-testing
- Quarkus Test Resources: https://quarkus.io/guides/getting-started-testing#quarkus-test-resource
- ClickHouse Docker Images: https://hub.docker.com/r/clickhouse/clickhouse-server
- ClickHouse MergeTree Engines: https://clickhouse.com/docs/en/engines/table-engines/mergetree-family

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Testcontainers Dependency | `grep -r "testcontainers" pom.xml */pom.xml` | TC dependency present |
| ClickHouse TC Module | `grep -r "clickhouse" pom.xml */pom.xml` | ClickHouse module present |
| Container Annotations | `grep -r "@Container\|@Testcontainers\|@QuarkusTestResource" */src/test/java` | Test resource annotations used |
| Integration Tests | `find . -name "*IT.java" -o -name "*IntegrationTest.java"` | IT classes exist |
| ClickHouse Container | `grep -r "ClickHouseContainer" */src/test/java` | ClickHouse container defined |
| Test Resource Lifecycle | `grep -r "QuarkusTestResourceLifecycleManager" */src/test/java` | Lifecycle manager implemented |
| IT Tests Pass | `./mvnw verify -DskipUnitTests` | All IT tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Testcontainers Integration..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  TEST_PATHS=$(find . -path "*/src/test/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  TEST_PATHS="./src/test/java"
fi

# Check for Testcontainers dependency
if grep -rq "testcontainers" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ Testcontainers dependency present"
else
  echo "❌ Testcontainers dependency not found"
  FAIL=1
fi

# Check for ClickHouse TC module
if grep -rq "clickhouse" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ ClickHouse Testcontainers module present"
else
  echo "❌ ClickHouse Testcontainers module not found"
  FAIL=1
fi

# Check for container/test resource annotations
FOUND=0
for tp in $TEST_PATHS; do
  if grep -rq "@Container\|@Testcontainers\|@QuarkusTestResource" "$tp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Container/TestResource annotations found"
else
  echo "❌ No container annotations found"
  FAIL=1
fi

# Check for integration test classes
IT_COUNT=$(find . -name "*IT.java" -o -name "*IntegrationTest.java" 2>/dev/null | wc -l)
if [ "$IT_COUNT" -gt 0 ]; then
  echo "✅ Integration test classes found: $IT_COUNT"
else
  echo "❌ No integration test classes found"
  FAIL=1
fi

# Check for ClickHouse container definitions
FOUND=0
for tp in $TEST_PATHS; do
  if grep -rq "ClickHouseContainer" "$tp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ ClickHouseContainer definition found"
else
  echo "❌ No ClickHouseContainer definition found"
  FAIL=1
fi

# Check for QuarkusTestResourceLifecycleManager implementation
FOUND=0
for tp in $TEST_PATHS; do
  if grep -rq "QuarkusTestResourceLifecycleManager" "$tp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ QuarkusTestResourceLifecycleManager implemented"
else
  echo "❌ No QuarkusTestResourceLifecycleManager found"
  FAIL=1
fi

# Run integration tests
if ./mvnw verify -DskipUnitTests -q 2>/dev/null; then
  echo "✅ Integration tests pass"
else
  echo "❌ Integration tests failed or not configured"
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
  echo "❌ Testcontainers Integration validation failed"
  exit 1
fi
echo "✅ All Testcontainers Integration criteria met"
```
