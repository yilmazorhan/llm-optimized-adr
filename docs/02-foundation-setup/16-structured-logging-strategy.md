# Issue: Structured Logging Strategy for Cloud-Native Applications

Unstructured logging in distributed systems causes: (1) increased Mean Time To Resolution (MTTR), (2) failure of automated log analysis pipelines, (3) non-compliance with audit trail requirements, and (4) inability to correlate events across service boundaries.

# Decision:

Implement comprehensive structured JSON logging with OpenTelemetry correlation, standardized formats, correlation ID propagation, sensitive data redaction, and centralized utilities.

# Constraints:

- **Log Format**: Production (`%prod`) **MUST** use JSON output; development (`%dev`) **MAY** use plaintext
- **Timestamps**: **MUST** use ISO 8601 format `uuuu-MM-dd'T'HH:mm:ss.SSSXXX`
- **Framework**: Applications **MUST** use Quarkus JBoss Log Manager + SLF4J facade only
- **MDC Fields**: Every log event **MUST** include `traceId`, `spanId`, `correlationId`
- **Redaction**: Values matching `password`, `secret`, `token`, `apiKey`, `authorization`, `credential` **MUST** be sanitized before logging
- **Output**: Logs **MUST** be written to stdout only; output **MUST** be parseable by Fluent Bit, Loki, ELK
- **Reactive**: MDC context **MUST** propagate via SmallRye across Mutiny async boundaries
- **Compliance**: Logs **MUST NOT** contain PII (GDPR) or PHI (HIPAA)
- **Prohibited**: `System.out`, `System.err`, `printStackTrace()` **MUST NOT** appear in production code
- **Log Purpose**: Each log statement **MUST** serve one of: debug, audit, error, operational. Redundant logs **MUST NOT** be added

# Alternatives:

**Text-based Logging**: Unstructured plain text requires regex-based parsing for every query. Fluent Bit, Grafana Loki, and Elasticsearch all document JSON as the recommended input format for zero-config field extraction (https://docs.fluentbit.io/manual/pipeline/parsers/json, https://grafana.com/docs/loki/latest/send-data/promtail/pipelines/). Regex parsers are fragile when log message formats change and cannot extract dynamic MDC fields. Rejected because the project requires automated field-level querying across `traceId`, `correlationId`, and custom MDC keys.

**Log4j2 JSON Layout**: Log4j2 provides `JsonLayout` and `JsonTemplateLayout` for structured output (https://logging.apache.org/log4j/2.x/manual/json-template-layout.html). However, Quarkus uses JBoss Log Manager as its built-in logging backend; adding Log4j2 requires excluding the default backend and bridging SLF4J, which the Quarkus logging guide explicitly discourages (https://quarkus.io/guides/logging#logging-apis). The `quarkus-logging-json` extension provides equivalent JSON output without framework conflicts. Rejected because the additional dependency and bridging configuration provide no benefit over the native Quarkus logging JSON extension.

# Rationale:

Quarkus JBoss Log Manager writes structured JSON when `quarkus.log.console.json=true` is set, emitting each log event as a single JSON object with `timestamp`, `level`, `loggerName`, and `message` fields (https://quarkus.io/guides/logging#json-logging). MDC values added via `MDC.put()` are included as top-level JSON fields, enabling Loki LogQL and Elasticsearch KQL to filter by `correlationId` or `traceId` without parsing the message body. The SLF4J facade is the logging API supported by Quarkus for GraalVM native compilation; alternative frameworks require reflection configuration and bridging adapters (https://quarkus.io/guides/logging#logging-apis). SmallRye Context Propagation captures MDC values before Mutiny async boundaries and restores them on the executing thread (https://smallrye.io/smallrye-mutiny/latest/guides/context-passing/), ensuring correlation IDs persist across reactive chains.

# Implementation Guidelines:

## Naming Conventions:
- Log category names: `com.example.[module].[component]` (e.g., `com.example.user.service`)
- MDC keys: camelCase (e.g., `correlationId`, `traceId`, `spanId`, `userId`)
- Log message format: `[Action] [Entity/Context]: [Details]` (e.g., `Creating user: email=john@example.com`)
- Filter classes: `[Purpose]RequestFilter` (e.g., `LoggingRequestFilter`)
- Context providers: `[Purpose]Context` (e.g., `LogContext`)

## Application Configuration:
```properties
# Production JSON logging
%prod.quarkus.log.console.json=true
%prod.quarkus.log.console.json.date-format=uuuu-MM-dd'T'HH:mm:ss.SSSXXX
%prod.quarkus.log.console.json.pretty=false
%prod.quarkus.log.console.json.exception-output-type=formatted

# Development text logging
%dev.quarkus.log.console.json=false
```

## Correlation Filter:
```java
@Provider
@Priority(Priorities.AUTHENTICATION)
public class LoggingRequestFilter implements ContainerRequestFilter, ContainerResponseFilter {
  @Override
  public void filter(ContainerRequestContext requestContext) {
    String header = requestContext.getHeaderString("X-Correlation-Id");
    String correlationId = header == null || header.isBlank() ? UUID.randomUUID().toString() : header;
    MDC.put("correlationId", correlationId);
    
    Span current = Span.current();
    SpanContext sc = current.getSpanContext();
    if (sc.isValid()) {
      MDC.put("traceId", sc.getTraceId());
      MDC.put("spanId", sc.getSpanId());
    }
  }

  @Override
  public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) {
    responseContext.getHeaders().putSingle("X-Correlation-Id", MDC.get("correlationId"));
  }
}
```

## Reactive Context:
```java
@ApplicationScoped
public class LogContext {
  public <T> Uni<T> with(String key, Object value, Supplier<Uni<T>> action) {
    return Uni.createFrom().item(() -> {
          MDC.put(key, String.valueOf(value));
          return null;
        })
        .flatMap(ignored -> action.get())
        .eventually(() -> MDC.remove(key));
  }
}
```

## Security Implementation:

**Sensitive Data Sanitizer**: Implement `SensitiveValueSanitizer` that replaces regex patterns `(?i)password|secret|token|authorization` with `***`, hashes long secrets using SHA-256 truncated format when logging necessary, sanitizes before any log output occurs.

**Code Quality**: Add Checkstyle rules to forbid `System.out` usage in production code, prevent `printStackTrace` calls that bypass structured logging.

## Usage Documentation:
```bash
# Example correlation ID usage
curl -H "X-Correlation-Id: demo-123" http://localhost:8080/api/users
```

## Validation:
```bash
# Verify JSON logging in production
./mvnw quarkus:dev -Dquarkus.profile=prod

# Validate structured log format
curl -X GET http://localhost:8080/health | jq '.timestamp,.level,.correlationId'

# Test correlation propagation
curl -H "X-Correlation-Id: test-correlation-123" http://localhost:8080/api/users
grep "test-correlation-123" logs/application.log | jq '.'

# Run integration tests
./mvnw test -Dtest=*LoggingIntegrationTest*
```

# Additional Recommendations:

- It is RECOMMENDED to adopt the OpenTelemetry Log Bridge when it reaches stable status to unify spans and logs in a single pipeline (https://opentelemetry.io/docs/specs/otel/logs/bridge-api/)
- It is RECOMMENDED to monitor log volume via Prometheus `log_events_total` metric and implement debug sampling if storage costs exceed thresholds (https://quarkus.io/guides/micrometer)
- Redaction patterns **SHOULD** be reviewed periodically against GDPR (https://gdpr.eu/article-25-data-protection-by-design/) and HIPAA requirements
- It is RECOMMENDED to maintain a catalog of domain-specific MDC keys with consistent camelCase naming across all services (see ADR-11, ADR-19)
- For non-containerized environments, it is RECOMMENDED to configure file-based log rotation via `quarkus.log.file.*` properties (https://quarkus.io/guides/logging#logging-configuration-reference)
- It is RECOMMENDED to include logging overhead in performance benchmarks using Quarkus continuous testing (https://quarkus.io/guides/continuous-testing)


# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| JSON Logging Config | `grep "quarkus.log.console.json" application.properties` | JSON format enabled for prod |
| Log Format | Application log output in prod profile | Valid JSON structure |
| Correlation ID | Log entries | `correlationId` field present |
| Trace Context | Log entries | `traceId`/`spanId` fields present |
| Log Levels Configured | `grep "quarkus.log.level" application.properties` | Levels explicitly set |
| No Sensitive Data | Log inspection | No passwords/tokens in logs |
| No System.out | `grep -r "System.out" src/main/java` | Zero matches |
| No printStackTrace | `grep -r "printStackTrace" src/main/java` | Zero matches |
| Checkstyle Passes | `./mvnw checkstyle:check` | Exit code 0 |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Structured Logging Strategy..."
FAIL=0

# Check for JSON logging configuration
if grep -rq "quarkus.log.console.json=true" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ JSON logging configured"
else
  echo "❌ JSON logging not configured"
  FAIL=1
fi

# Check for log level configuration
if grep -rq "quarkus.log.level" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ Log levels configured"
else
  echo "❌ Log levels not explicitly configured"
  FAIL=1
fi

# Check for category-specific log levels
if grep -rq "quarkus.log.category" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ Category-specific logging configured"
else
  echo "❌ Category-specific logging not configured"
  FAIL=1
fi

# Check for trace context in JSON logs
if grep -rq "additional-field.*traceId\|additional-field.*correlationId" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ Trace context fields configured in JSON logs"
else
  echo "❌ Trace context not configured in JSON logs"
  FAIL=1
fi

# Look for MDC/correlation ID usage in code
if grep -rq "MDC\|correlationId\|traceId" */src/main/java 2>/dev/null; then
  echo "✅ Correlation ID/MDC usage found"
else
  echo "❌ No MDC/correlation ID usage detected"
  FAIL=1
fi

# Check for System.out/err usage (forbidden)
SYSOUT=$(grep -rn "System\.out\|System\.err" */src/main/java 2>/dev/null | wc -l)
if [ "$SYSOUT" -eq 0 ]; then
  echo "✅ No System.out/err usage detected"
else
  echo "❌ System.out/err usage found: $SYSOUT occurrences"
  FAIL=1
fi

# Check for printStackTrace usage (forbidden)
STACKTRACE=$(grep -rn "\.printStackTrace()" */src/main/java 2>/dev/null | wc -l)
if [ "$STACKTRACE" -eq 0 ]; then
  echo "✅ No printStackTrace() usage detected"
else
  echo "❌ printStackTrace() usage found: $STACKTRACE occurrences"
  FAIL=1
fi

# Check for sensitive data patterns in log statements
SENSITIVE=$(grep -rn "log.*password\|log.*secret\|log.*token" */src/main/java 2>/dev/null | wc -l)
if [ "$SENSITIVE" -eq 0 ]; then
  echo "✅ No sensitive data logging detected"
else
  echo "❌ Potential sensitive data logging found: $SENSITIVE occurrences"
  FAIL=1
fi

# Build verification
./mvnw clean compile -q && echo "✅ Build successful" || { echo "❌ Build failed"; FAIL=1; }

if [ $FAIL -ne 0 ]; then
  echo "❌ Structured Logging Strategy validation FAILED"
  exit 1
fi
echo "✅ All Structured Logging Strategy criteria met"
```
