# ADR-19: Audit Logging Strategy

# Issue:

Applications must log significant events for security, compliance, and forensic analysis. Without a standard, audit trails are inconsistent, difficult to analyze, and may not meet regulatory requirements (e.g., SOC 2, HIPAA, GDPR). This increases security risks and complicates incident response.

# Decision:

A structured, asynchronous audit logging mechanism **MUST** be implemented. All security-sensitive operations and significant business events **MUST** be logged.

### Key Requirements
- **Format**: Logs **MUST** be structured JSON, following a standard schema.
- **Execution**: Logging **MUST** be asynchronous to minimize performance impact.
- **Destination**: Logs **MUST** be written to a dedicated, isolated stream, separate from application logs.
- **Data Collection**: An "Audit Context" **MUST** collect relevant information throughout a request's lifecycle.
- **Immutability**: Log entries **MUST** be immutable once written.

# Constraints:

- Audit log events **MUST** add less than 5ms of latency to any request.
- Audit log delivery **MUST** be guaranteed using a persistent queue or reliable agent.
- The audit trail **MUST** be protected from unauthorized access; sensitive data (PII) **MUST** be masked or encrypted within logs.
- Audit records **MUST NOT** be lost during application shutdown or crashes.

# Alternatives:

### 1. Synchronous Logging

Synchronous audit logging blocks the calling thread until the log entry is written. According to the Quarkus Reactive Architecture guide (https://quarkus.io/guides/mutiny-primer), blocking operations on the I/O thread cause `BlockingNotAllowedException` and degrade throughput. Benchmarks in high-concurrency environments show synchronous file I/O adding 10–50ms per call depending on disk latency (https://openjdk.org/jeps/425), which exceeds the 5ms latency constraint defined in this ADR.

### 2. Database-Only Auditing (Direct Table Insert)

Writing audit records directly to a database couples audit write performance to database throughput. According to ClickHouse performance benchmarks (https://clickhouse.com/benchmark/dbms/), analytical databases optimize for batch inserts rather than single-row writes. Direct INSERT per request under load creates lock contention and connection pool exhaustion, as documented in the HikariCP wiki (https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing). This approach also violates the isolation requirement since audit writes share the same connection pool as business operations.

# Rationale:

Asynchronous structured JSON logging decouples audit event generation from the business transaction by offloading serialization and I/O to a worker thread via `Infrastructure.getDefaultWorkerPool()`, as prescribed by the Mutiny infrastructure guide (https://smallrye.io/smallrye-mutiny/latest/guides/emit-on/). This ensures the calling reactive pipeline remains non-blocking.

Dedicated log streams written to a separate file appender, configured through Quarkus logging handlers (https://quarkus.io/guides/logging#file-log-handler), provide physical isolation from application logs. This separation enables independent log rotation, retention policies, and forwarding to SIEM systems without affecting application log processing.

The JSON schema enforces a consistent structure across all services, enabling automated parsing and alerting. Javers diff generation (https://javers.org/documentation/diff-examples/) produces deterministic change sets for UPDATE events, eliminating manual diff construction errors.

# Implementation Guidelines:

### 1. Add Dependencies

To generate object diffs, use a library like Javers.

```xml
<dependency>
    <groupId>org.javers</groupId>
    <artifactId>javers-core</artifactId>
    <version>7.3.2</version>
</dependency>
```

### 2. Define the Audit Log JSON Schema

The standard audit log entry must contain the following fields:

```json
{
  "id": "evt_2a86bde0-9c3b-4b1b-9b4a-3a2b8c7d6e5f",
  "timestamp": "2025-09-29T10:30:00.123Z",
  "serviceName": "user-service",
  "serviceVersion": "1.2.3",
  "eventSource": {
    "component": "api.v1.users",
    "host": "user-service-7c7f-b4f6"
  },
  "eventName": "USER_UPDATED",
  "outcome": "SUCCESS",
  "correlationId": "corr_a1b2c3d4",
  "traceId": "trace_e5f6g7h8",
  "actor": {
    "type": "USER",
    "id": "usr_a1b2c3d4e5f6",
    "ipAddress": "192.168.1.100",
    "portNumber": 8080
  },
  "resource": {
    "type": "USER",
    "id": "usr_f6e5d4c3b2a1"
  },
  "eventData": {
    "userId": "usr_f6e5d4c3b2a1"
  },
  "diff": [
    {
      "type": "VALUE_CHANGED",
      "property": "firstName",
      "oldValue": "John",
      "newValue": "Jonathan"
    },
    {
      "type": "VALUE_CHANGED",
      "property": "lastName",
      "oldValue": "Doe",
      "newValue": "Doeman"
    }
  ]
}
```

#### Schema Field Descriptions
- **`id`**: Unique event identifier (`evt_<UUID>`).
- **`timestamp`**: ISO 8601 UTC timestamp of the event.
- **`serviceName`**: Microservice name from `quarkus.application.name`.
- **`serviceVersion`**: Microservice version from `quarkus.application.version`.
- **`eventSource.component`**: Architectural component that generated the event (e.g., `domain.user.UserService`).
- **`eventSource.host`**: Hostname or container ID of the service instance.
- **`eventName`**: Standardized event name (`RESOURCE_ACTION`, e.g., `USER_CREATED`).
- **`outcome`**: Result of the event (`SUCCESS` or `FAILURE`).
- **`correlationId`**: Identifier to correlate logs within a single request. Must match the ID in application logs (ADR-16).
- **`traceId`**: OpenTelemetry distributed trace identifier (ADR-20).
- **`actor.type`**: Type of actor (`USER`, `SYSTEM`, `API_KEY`).
- **`actor.id`**: Unique identifier for the actor (e.g., JWT `sub` claim).
- **`actor.ipAddress`**: Source IP address of the request.
- **`actor.portNumber`**: Source port of the request.
- **`resource.type`**: Type of resource affected (e.g., `USER`, `ORDER`).
- **`resource.id`**: Unique identifier for the resource instance.
- **`eventData`**: JSON object with relevant event data. Sensitive data must be masked.
- **`diff`**: Array of change objects for `UPDATED` events. Null for `CREATED` or `DELETED`.
  - `type`: Type of change (`VALUE_CHANGED`, `ELEMENT_ADDED`).
  - `property`: Name of the changed property.
  - `oldValue`: Value before the change (redact if sensitive).
  - `newValue`: Value after the change (redact if sensitive).

### Domain Model Contract

To prevent partial or ad-hoc audit models, every service **must** expose a strongly typed `AuditLog` aggregate mirroring the JSON schema above. The contract is non-negotiable:

- Top-level fields (`id`, `timestamp`, `serviceName`, `serviceVersion`, `eventName`, `outcome`, `correlationId`, `traceId`) are required constructor arguments. Nothing may default to `null`.
- Nested value objects must exist for `eventSource`, `actor`, `resource`, and `diff` entries. Use dedicated types (records/classes) rather than raw `Map` or `JsonObject` structures to enforce presence and masking rules.
- `actor` must support multiple performer types (human user, system integration, API key). Do **not** overload `resource` or `eventData` to store actor identifiers.
- `eventData` should be modeled as a purpose-built DTO per event family so that sensitive fields can be masked deterministically before serialization.
- `diff` collections are optional only for create/delete events; when present, each element must capture `type`, `property`, `oldValue`, and `newValue` (with redaction policies applied).

> :warning: Implementations that omit any field or collapse nested structures into a generic `attributes` map are non-compliant and must not pass review.

### 3. Create the Audit Logging Service

This service creates and sends audit events to a dedicated logger.

```java
package com.copilot.quarkus.infrastructure.audit;

import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.json.bind.Jsonb;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@ApplicationScoped
public class AuditLogService {

    // A dedicated logger for audit events
    private static final Logger AUDIT_LOGGER = LoggerFactory.getLogger("AuditLogger");

    @Inject
    Jsonb jsonb;

    public Uni<Void> logEvent(AuditLog log) {
        return Uni.createFrom().item(log)
            .emitOn(Infrastructure.getDefaultWorkerPool()) // Execute on a worker thread
            .invoke(auditLog -> {
                String jsonLog = jsonb.toJson(auditLog);
                AUDIT_LOGGER.info(jsonLog);
            })
            .onFailure().invoke(ex -> {
                // Fallback logging if JSON serialization fails
                LoggerFactory.getLogger(AuditLogService.class)
                             .error("Failed to serialize audit log event: {}", log, ex);
            })
            .replaceWithVoid();
    }
}
```

### 4. Configure the Dedicated Audit Logger

Configure a separate file appender for the audit logger in `application.properties`.

```properties
# Audit Logging Configuration
quarkus.log.category."AuditLogger".level=INFO
# Disable logging to the console for the audit logger
quarkus.log.category."AuditLogger".use-parent-handlers=false

# Configure a separate file handler for audit logs
quarkus.log.handler.file."audit".enable=true
quarkus.log.handler.file."audit".path=/var/log/app/audit.log
quarkus.log.handler.file."audit".level=INFO
# Use a pattern formatter that only outputs the message itself
quarkus.log.handler.file."audit".formatter.pattern=%s%n

# Assign the file handler to the audit logger
quarkus.log.logger."AuditLogger".handlers=audit
```

### 5. Integrate with Business Logic

Inject and use the `AuditLogService` in application services.

```java
package com.copilot.quarkus.domain.user.service;

import com.copilot.quarkus.domain.user.model.User;
import com.copilot.quarkus.infrastructure.audit.AuditLog;
import com.copilot.quarkus.infrastructure.audit.AuditLogService;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import org.javers.core.Javers;
import org.javers.core.diff.Diff;

@ApplicationScoped
public class UserService {

    @Inject
    AuditLogService auditLogService;
    
    @Inject
    Javers javers;

    // ... other dependencies

    public Uni<User> updateUser(UpdateUserCommand command) {
        return userRepository.findById(command.getUserId())
            .onItem().ifNull().failWith(new UserNotFoundException(command.getUserId()))
            .flatMap(existingUser -> {
                User oldUser = existingUser.clone(); // Create a snapshot for diffing

                // Apply updates
                existingUser.updateFirstName(command.getFirstName());
                existingUser.updateLastName(command.getLastName());

                return userRepository.save(existingUser)
                    .call(updatedUser -> {
                        Diff diff = javers.compare(oldUser, updatedUser);
                        AuditLog log = new AuditLog.Builder()
                            .withEventName("USER_UPDATED")
                            // ... other fields
                            .withDiff(diff.getChanges())
                            .build();
                        return auditLogService.logEvent(log);
                    });
            });
    }
}
```

# Additional Recommendations:

- **RECOMMENDED**: Use **Javers** for automated diff generation instead of manual diff construction — see Javers diff documentation (https://javers.org/documentation/diff-examples/).
- **RECOMMENDED**: Use a request-scoped CDI bean to propagate audit context (`correlationId`, `traceId`, actor details) through the call stack — see Quarkus CDI reference (https://quarkus.io/guides/cdi-reference).
- **RECOMMENDED**: Configure log rotation for `audit.log` and forward via **Fluentd** or **Filebeat** to a centralized SIEM — see Fluentd architecture (https://docs.fluentd.org/architecture/life-of-a-fluentd-event) and Filebeat overview (https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-overview.html).
- **RECOMMENDED**: Create a PII masking utility to redact sensitive fields in `eventData` and `diff` before serialization — see OWASP Logging Cheat Sheet (https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html).
- **RECOMMENDED**: Write integration tests asserting audit log output using an in-memory appender or temporary file — see Quarkus testing guide (https://quarkus.io/guides/getting-started-testing).

# Success Criteria:

**Completion Gate**: The following criteria **MUST** all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| AuditLog Entity | `grep -r "class AuditLog" */src/main/java` | Strongly-typed AuditLog aggregate exists |
| AuditLogService | `grep -r "AuditLogService" */src/main/java` | Async logging service with Mutiny worker pool |
| Dedicated Logger Config | `grep "AuditLogger" */src/main/resources/application.properties` | Separate file handler for audit events |
| Javers Dependency | `grep "javers-core" */pom.xml` | Diff generation library present |
| Async Execution | `grep -r "getDefaultWorkerPool\|emitOn" */src/main/java` | Non-blocking audit I/O |
| PII Masking | `grep -r "mask\|redact" */src/main/java` | Sensitive data protection utility |
| Audit Tests | `./mvnw test -Dtest=*Audit*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Audit Logging Strategy..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for AuditLog or AuditEvent class
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "class AuditLog\|class AuditEvent" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Audit log entity class found"
else
  echo "❌ No AuditLog or AuditEvent class found"
  FAIL=1
fi

# Check for AuditLogService
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "AuditLogService\|AuditEventService" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Audit logging service found"
else
  echo "❌ No AuditLogService found"
  FAIL=1
fi

# Check for dedicated audit logger configuration
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "AuditLogger\|handler.file.\"audit\"" "$rp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Dedicated audit logger configuration found"
else
  echo "❌ No dedicated audit logger configuration in application.properties"
  FAIL=1
fi

# Check for Javers dependency
if find . -name "pom.xml" -exec grep -q "javers-core" {} + 2>/dev/null; then
  echo "✅ Javers dependency found for diff generation"
else
  echo "❌ Javers dependency not found in pom.xml"
  FAIL=1
fi

# Check for async execution pattern
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "getDefaultWorkerPool\|emitOn\|runSubscriptionOn" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Asynchronous execution pattern found"
else
  echo "❌ No async execution pattern (emitOn/getDefaultWorkerPool) in audit logging"
  FAIL=1
fi

# Check for PII masking utility
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "mask\|redact\|sanitize\|PiiMask" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ PII masking utility found"
else
  echo "❌ No PII masking utility found"
  FAIL=1
fi

# Run audit tests
if ./mvnw test -Dtest="*Audit*" -q 2>/dev/null; then
  echo "✅ Audit tests pass"
else
  echo "❌ Audit tests failed or not found"
  FAIL=1
fi

# Build verification
if ./mvnw clean verify -q 2>/dev/null; then
  echo "✅ Build succeeds"
else
  echo "❌ Build failed"
  FAIL=1
fi

if [ "$FAIL" -ne 0 ]; then
  echo "❌ Audit Logging Strategy validation failed"
  exit 1
fi
echo "✅ All Audit Logging Strategy criteria met"
```
