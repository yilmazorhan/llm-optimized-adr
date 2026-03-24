# Issue: Exception Handling Strategy for Reactive Applications

Quarkus microservices with hexagonal design require robust exception handling addressing reactive programming complexity, distributed system error propagation, and observability requirements.

# Decision:

Implement reactive-first exception handling with structured error context using Quarkus fault tolerance capabilities and OpenTelemetry integration. All applications enforce layered exception strategies maintaining hexagonal architecture boundaries while providing comprehensive error observability.

**Required Exception Architecture**:
- Layered exception hierarchy with domain/infrastructure/application separation
- Reactive error handling patterns with Mutiny integration
- Structured error context with correlation IDs and business context
- OpenTelemetry span integration for distributed tracing
- Circuit breaker and retry patterns for resilience
- Memory-efficient error context collection
- GraalVM native compilation compatibility
- Centralized error code registry with remediation guidance

# Constraints:

- Error context propagation **MUST NOT** introduce blocking operations in reactive streams
- Exception transformations **MUST** maintain reactive stream semantics without breaking backpressure
- Error handling **MUST** integrate with Mutiny patterns without blocking thread pools
- Context propagation **MUST** work correctly across thread boundaries in async operations
- Domain exceptions **MUST** remain pure without infrastructure dependencies
- Adapter layer **MUST** handle all external system exception translation
- Port interfaces **MUST** clearly define exception contracts
- All exception handling **MUST** be compatible with GraalVM native compilation
- Error context objects **MUST** be registered for reflection in native builds via `@RegisterForReflection`
- Throwing `new RuntimeException(...)` or `new Exception(...)` directly **MUST NOT** appear in production code
- All custom exceptions **MUST** carry unique, documented error codes (e.g., `DOM-001`, `INF-001`, `AUD-001`)
- All exceptions **MUST** include operation context, correlation ID, and sufficient details for debugging


# Alternatives:

**Generic Exception with Stack Traces**: Relying on `RuntimeException` / `Exception` and stack-trace inspection to diagnose errors. The Java Language Specification §11.1.1 defines unchecked exceptions as a mechanism the compiler does not verify (https://docs.oracle.com/javase/specs/jls/se21/html/jls-11.html#jls-11.1.1), so untyped exceptions provide no compile-time contract for callers. Stack traces are expensive—`Throwable.fillInStackTrace()` walks the entire call chain on every construction (https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Throwable.html#fillInStackTrace())—and expose internal class paths in API responses (CWE-209, https://cwe.mitre.org/data/definitions/209.html). Rejected because the project’s hexagonal architecture requires typed port-level exception contracts and structured error codes.

**MicroProfile Fault Tolerance Only (no custom hierarchy)**: SmallRye Fault Tolerance provides `@Retry`, `@CircuitBreaker`, `@Fallback`, and `@Timeout` annotations (https://download.eclipse.org/microprofile/microprofile-fault-tolerance-4.1/microprofile-fault-tolerance-spec-4.1.html). These annotations handle transient failures but do not define domain-versus-infrastructure exception taxonomy and do not attach structured `ErrorContext` with correlation IDs. Rejected as the sole strategy because fault-tolerance annotations complement, but do not replace, a typed exception hierarchy.

# Rationale:

Mutiny’s `Uni.onFailure()` and `Multi.onFailure()` operators propagate exceptions without blocking the event loop, preserving the non-blocking guarantee documented in the Mutiny specification (https://smallrye.io/smallrye-mutiny/latest/guides/handling-failures/). Typed domain exceptions (`DomainException`, `InfrastructureException`) enforce the hexagonal architecture port boundary: adapter code catches infrastructure-specific exceptions (e.g., `SQLException`) and translates them to port-level types, preventing infrastructure leakage into the domain layer (see ADR-10). Each exception carries an `ErrorContext` with a unique error code and correlation ID, enabling Grafana Loki / Elasticsearch field-level queries without stack-trace parsing. SmallRye Fault Tolerance annotations (`@Retry`, `@CircuitBreaker`, `@Timeout`) complement the hierarchy by handling transient infrastructure failures declaratively (https://quarkus.io/guides/smallrye-fault-tolerance). `@RegisterForReflection` on exception classes ensures GraalVM native-image can instantiate them at runtime (https://quarkus.io/guides/writing-native-applications-tips#registerForReflection).

# Implementation Guidelines:

## Mandatory Rules

> **CRITICAL**: The following rules are **non-negotiable** and **MUST** be enforced across all application code.

1. **No Raw RuntimeException**: Throwing `new RuntimeException(...)` is **STRICTLY PROHIBITED**. All exceptions **MUST** be wrapped in domain-specific or infrastructure-specific exception types.
2. **No Generic Exception**: Throwing `new Exception(...)` or catching generic `Exception` without re-wrapping is **PROHIBITED**.
3. **Layer-Appropriate Exceptions**: Domain layer **MUST** throw `DomainException` subclasses. Infrastructure layer **MUST** throw `InfrastructureException` subclasses.
4. **Meaningful Error Codes**: All custom exceptions **MUST** have unique, documented error codes (e.g., `DOM-001`, `INF-001`, `AUD-001`).
5. **Exception Context Required**: All exceptions **MUST** include operation context, correlation ID, and sufficient details for debugging.

## Pre-Implementation Checklist

> Before throwing ANY exception in this project, verify:

- [ ] **Never throw RuntimeException** — create a specific exception class instead
- [ ] **Check existing exception types** — review if an appropriate exception already exists
- [ ] **Create domain-specific exception** — extend `DomainException` or `InfrastructureException`
- [ ] **Include error code** — add unique code like `AUD-001`, `INF-002`
- [ ] **Add context** — include relevant IDs, operation name, and failure reason
- [ ] **Document in error registry** — add new exception to centralized error documentation

### Quick Reference: Correct vs Incorrect

| ❌ PROHIBITED | ✅ REQUIRED |
|---------------|-------------|
| `throw new RuntimeException("Failed to save", e)` | `throw new AuditPersistenceException("save", eventId, e)` |
| `throw new RuntimeException(e.getMessage())` | `throw new DatabaseConnectionException(operation, e)` |
| `throw new Exception("Something went wrong")` | `throw new EntityNotFoundException("User", userId)` |
| `catch (Exception e) { log(e); }` | `catch (SQLException e) { throw new DatabaseException(op, e); }` |

**Domain Layer Exception Base**:
```java
@RegisterForReflection // GraalVM native compilation support
public abstract class DomainException extends RuntimeException {
    private final ErrorContext errorContext;
    
    protected DomainException(String errorCode, String operation, String reason, Map<String, Object> context) {
        super(String.format("[%s] Operation '%s' failed: %s", errorCode, operation, reason));
        this.errorContext = ErrorContext.builder()
            .errorCode(errorCode)
            .operationName(operation)
            .correlationId(MDC.get("correlationId"))
            .timestamp(Instant.now())
            .businessContext(context)
            .build();
    }
    
    public ErrorContext getErrorContext() {
        return errorContext;
    }
}

// Business Rule Violations
public class UserEmailAlreadyExistsException extends DomainException {
    public UserEmailAlreadyExistsException(String email) {
        super("DOM-001", "CreateUser", 
              String.format("Email '%s' already exists in system", email),
              Map.of("email", email, "suggestion", "Use different email or retrieve existing user"));
    }
}

// Entity Not Found
public class EntityNotFoundException extends DomainException {
    public EntityNotFoundException(String entityType, String entityId) {
        super("DOM-002", "FindEntity",
              String.format("%s with ID '%s' not found", entityType, entityId),
              Map.of("entityType", entityType, "entityId", entityId));
    }
}
```

**Infrastructure Layer Exception Translation**:
```java
@ApplicationScoped
public abstract class InfrastructureException extends RuntimeException {
    private final ErrorContext errorContext;
    
    protected InfrastructureException(String errorCode, String operation, String reason, Throwable cause) {
        super(String.format("[%s] Infrastructure operation '%s' failed: %s", errorCode, operation, reason), cause);
        this.errorContext = ErrorContext.builder()
            .errorCode(errorCode)
            .operationName(operation)
            .correlationId(MDC.get("correlationId"))
            .timestamp(Instant.now())
            .technicalContext(Map.of("originalError", cause.getMessage(), "errorType", cause.getClass().getSimpleName()))
            .build();
    }
}

// Database Adapter Exceptions
public class DatabaseConnectionException extends InfrastructureException {
    public DatabaseConnectionException(String operation, SQLException cause) {
        super("INF-001", operation, 
              String.format("Database connection failed: %s", cause.getMessage()), cause);
    }
}

// External Service Exceptions  
public class ExternalServiceException extends InfrastructureException {
    public ExternalServiceException(String serviceName, String operation, Exception cause) {
        super("INF-002", operation,
              String.format("External service '%s' unavailable: %s", serviceName, cause.getMessage()), cause);
    }
}

// Audit Persistence Exceptions (ClickHouse)
public class AuditPersistenceException extends InfrastructureException {
    public AuditPersistenceException(String operation, UUID eventId, SQLException cause) {
        super("AUD-001", operation,
              String.format("Audit persistence failed for event '%s': %s", eventId, cause.getMessage()), cause);
    }
    
    public AuditPersistenceException(String operation, int batchSize, SQLException cause) {
        super("AUD-002", operation,
              String.format("Audit batch persistence failed for %d events: %s", batchSize, cause.getMessage()), cause);
    }
}

// Audit Query Exceptions
public class AuditQueryException extends InfrastructureException {
    public AuditQueryException(String operation, String queryContext, SQLException cause) {
        super("AUD-003", operation,
              String.format("Audit query failed: %s - %s", queryContext, cause.getMessage()), cause);
    }
}
```

### ❌ Anti-Pattern: Raw RuntimeException (PROHIBITED)

> **WARNING**: The following patterns are ALL PROHIBITED. Any code using these patterns MUST be rejected in code review.

```java
// ❌ DO NOT DO THIS - Raw RuntimeException is PROHIBITED
@ApplicationScoped
public class BadRepository {
    
    public Uni<Void> save(Entity entity) {
        try {
            // database operation
        } catch (SQLException e) {
            // ❌ WRONG: Throwing raw RuntimeException
            throw new RuntimeException("Failed to save entity", e);
        }
    }
    
    public Uni<Entity> findById(String id) {
        try {
            // database operation  
        } catch (SQLException e) {
            // ❌ WRONG: Losing context with generic exception
            throw new RuntimeException(e.getMessage());
        }
    }
}
```

### ✅ Correct Pattern: Domain/Infrastructure Exceptions

```java
// ✅ CORRECT - Use specific infrastructure exception
@ApplicationScoped
public class GoodRepository {
    
    public Uni<Void> save(AuditEvent event) {
        try {
            // database operation
        } catch (SQLException e) {
            // ✅ CORRECT: Specific exception with context
            throw new AuditPersistenceException("save", event.id(), e);
        }
    }
    
    public Uni<AuditEvent> findById(UUID id) {
        try {
            // database operation  
        } catch (SQLException e) {
            // ✅ CORRECT: Specific exception with operation context
            throw new AuditQueryException("findById", "id=" + id, e);
        }
    }
}
```

## Code Review Checklist for Exception Handling

> **Reviewers MUST reject PRs that violate these requirements.**

| Check | How to Verify | Action if Violated |
|-------|--------------|-------------------|
| No `throw new RuntimeException(...)` | Search for `throw new RuntimeException` | **REJECT PR** |
| No `throw new Exception(...)` | Search for `throw new Exception(` | **REJECT PR** |
| Uses domain-specific exceptions | Verify extends `DomainException` or `InfrastructureException` | **REJECT PR** |
| Has unique error code | Verify exception has `DOM-XXX`, `INF-XXX`, `AUD-XXX` | **REJECT PR** |
| Includes operation context | Verify constructor includes operation name | **REJECT PR** |
| Preserves original cause | Verify `cause` is passed to parent | **REJECT PR** |

## 2. Reactive Error Handling Patterns

**Mutiny Integration Service**:
```java
@ApplicationScoped
public class ReactiveErrorHandler {
    
    @Inject
    ErrorContextManager errorContextManager;
    
    @Inject
    StructuredErrorLogger errorLogger;
    
    public <T> Uni<T> handleWithRetryAndFallback(Uni<T> operation, String operationName, Supplier<T> fallback) {
        return operation
            .onFailure().transform(this::translateToBusinessException)
            .onFailure(BusinessRuleViolationException.class).failWith(ex -> ex) // Don't retry business rule violations
            .onFailure(InfrastructureException.class).retry()
                .withBackOff(Duration.ofMillis(100), Duration.ofSeconds(2))
                .expireIn(Duration.ofSeconds(30))
                .atMost(3)
            .onFailure().call(failure -> {
                errorLogger.logError(operationName, failure);
                return Uni.createFrom().voidItem();
            })
            .onFailure(InfrastructureException.class).recoverWithItem(fallback);
    }
    
    private Throwable translateToBusinessException(Throwable cause) {
        if (cause instanceof SQLException) {
            return new DatabaseConnectionException("DatabaseOperation", (SQLException) cause);
        } else if (cause instanceof ConnectException) {
            return new ExternalServiceException("UnknownService", "ServiceCall", (Exception) cause);
        }
        return cause;
    }
    
    public <T> Uni<T> withCircuitBreaker(Uni<T> operation, String operationName) {
        return operation
            .onFailure().transform(this::translateToBusinessException)
            .onFailure().call(failure -> 
                recordCircuitBreakerMetrics(operationName, failure))
            .onFailure().invoke(failure -> 
                errorLogger.logError(operationName, failure));
    }
    
    private Uni<Void> recordCircuitBreakerMetrics(String operation, Throwable failure) {
        // Record metrics for circuit breaker decision making
        MeterRegistry.globalRegistry
            .counter("circuit_breaker_failures", "operation", operation, "error_type", failure.getClass().getSimpleName())
            .increment();
        return Uni.createFrom().voidItem();
    }
}
```

**Circuit Breaker Integration with Fault Tolerance**:
```java
@ApplicationScoped
public class ResilientUserService {
    
    @Inject
    UserRepository userRepository;
    
    @Inject
    ReactiveErrorHandler errorHandler;
    
    @CircuitBreaker(successThreshold = 3, requestVolumeThreshold = 4, failureRatio = 0.75, delay = 5000)
    @Retry(maxRetries = 3, delay = 1000, delayUnit = ChronoUnit.MILLIS, maxDuration = 10000)
    @Timeout(value = 5000, unit = ChronoUnit.MILLIS)
    @Fallback(fallbackMethod = "findUserFallback")
    public Uni<User> findUserById(UserId userId) {
        return errorHandler.handleWithRetryAndFallback(
            userRepository.findById(userId),
            "FindUserById",
            () -> createEmptyUser(userId)
        );
    }
    
    public Uni<User> findUserFallback(UserId userId) {
        errorLogger.logFallbackExecution("FindUserById", userId.toString());
        return Uni.createFrom().item(createEmptyUser(userId));
    }
    
    private User createEmptyUser(UserId userId) {
        return User.builder()
            .id(userId)
            .email("fallback@system.local")
            .firstName("System")
            .lastName("Fallback")
            .build();
    }
}
```

## 3. Error Context and Correlation

**Error Context Model**:
```java
@RegisterForReflection
public class ErrorContext {
    private final String errorCode;
    private final String correlationId;
    private final String operationName;
    private final Map<String, Object> businessContext;
    private final Map<String, Object> technicalContext;
    private final List<String> remediationSteps;
    private final String documentationUrl;
    private final Instant timestamp;
    private final String serviceVersion;
    
    // Builder pattern for construction
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private Map<String, Object> businessContext = new HashMap<>(8);
        private Map<String, Object> technicalContext = new HashMap<>(8);
        private List<String> remediationSteps = new ArrayList<>(4);
        
        public Builder businessContext(Map<String, Object> context) {
            if (this.businessContext.size() + context.size() <= 10) {
                this.businessContext.putAll(context);
            }
            return this;
        }
        
        public Builder addRemediation(String step) {
            if (remediationSteps.size() < 5) {
                remediationSteps.add(step);
            }
            return this;
        }
        
        // Additional builder methods...
    }
}
```

**Context Propagation Manager**:
```java
@ApplicationScoped
public class ErrorContextManager {
    
    @Inject
    @ConfigProperty(name = "quarkus.application.version")
    String applicationVersion;
    
    public <T> Uni<T> withErrorContext(Uni<T> operation, String operationName, Map<String, Object> businessContext) {
        String correlationId = generateCorrelationId();
        
        return Uni.createFrom().item(() -> {
            // Set MDC for this reactive chain
            MDC.put("correlationId", correlationId);
            MDC.put("operation", operationName);
            MDC.put("timestamp", Instant.now().toString());
            return operation;
        })
        .chain(op -> op)
        .onTermination().invoke(() -> {
            MDC.clear();
        });
    }
    
    public String generateCorrelationId() {
        return UUID.randomUUID().toString().replace("-", "").substring(0, 12);
    }
    
    public ErrorContext createErrorContext(String errorCode, String operation, Map<String, Object> context) {
        return ErrorContext.builder()
            .errorCode(errorCode)
            .correlationId(MDC.get("correlationId"))
            .operationName(operation)
            .businessContext(context)
            .serviceVersion(applicationVersion)
            .timestamp(Instant.now())
            .documentationUrl(String.format("https://docs.example.com/errors/%s", errorCode))
            .build();
    }
}
```

## 4. OpenTelemetry and Structured Logging

**Tracing Error Handler**:
```java
@ApplicationScoped
public class TracingErrorHandler {
    
    @Traced(operationName = "error-handling")
    public void handleError(Throwable error, String operationName, Map<String, Object> context) {
        Span currentSpan = Span.current();
        
        // Set error status on span
        currentSpan.setStatus(StatusCode.ERROR, error.getMessage());
        currentSpan.recordException(error);
        
        // Add structured attributes
        currentSpan.setAttributes(Attributes.of(
            AttributeKey.stringKey("error.type"), error.getClass().getSimpleName(),
            AttributeKey.stringKey("operation.name"), operationName,
            AttributeKey.stringKey("correlation.id"), MDC.get("correlationId"),
            AttributeKey.booleanKey("error.recoverable"), isRecoverable(error)
        ));
        
        // Add business context as span attributes
        context.forEach((key, value) -> {
            if (value != null && key.length() < 50 && value.toString().length() < 200) {
                currentSpan.setAttribute(AttributeKey.stringKey("business." + key), value.toString());
            }
        });
        
        // Create error event
        currentSpan.addEvent("error.occurred", Attributes.of(
            AttributeKey.stringKey("error.message"), error.getMessage(),
            AttributeKey.longKey("error.timestamp"), Instant.now().toEpochMilli()
        ));
    }
    
    private boolean isRecoverable(Throwable error) {
        return error instanceof InfrastructureException && 
               !(error instanceof BusinessRuleViolationException);
    }
}
```

**Structured Error Logger**:
```java
@ApplicationScoped
public class StructuredErrorLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(StructuredErrorLogger.class);
    
    @Inject
    @ConfigProperty(name = "quarkus.application.version")
    String applicationVersion;
    
    public void logError(String operationName, Throwable error) {
        Map<String, Object> logEntry = Map.of(
            "event_type", "ERROR",
            "operation_name", operationName,
            "error_type", error.getClass().getSimpleName(),
            "error_message", error.getMessage(),
            "correlation_id", MDC.get("correlationId"),
            "timestamp", Instant.now(),
            "service_version", applicationVersion,
            "trace_id", Span.current().getSpanContext().getTraceId(),
            "span_id", Span.current().getSpanContext().getSpanId(),
            "recoverable", isRecoverable(error)
        );
        
        if (error instanceof DomainException) {
            DomainException domainError = (DomainException) error;
            ErrorContext context = domainError.getErrorContext();
            Map<String, Object> enrichedEntry = new HashMap<>(logEntry);
            enrichedEntry.put("error_code", context.getErrorCode());
            enrichedEntry.put("business_context", context.getBusinessContext());
            enrichedEntry.put("remediation_steps", context.getRemediationSteps());
            
            logger.error("Domain error occurred: {}", enrichedEntry);
        } else {
            logger.error("Technical error occurred: {}", logEntry, error);
        }
    }
    
    public void logFallbackExecution(String operationName, String context) {
        logger.warn("Fallback executed for operation: {} with context: {}", operationName, context);
    }
    
    private boolean isRecoverable(Throwable error) {
        return error instanceof InfrastructureException && 
               !(error instanceof BusinessRuleViolationException);
    }
}
```

## 5. Application Configuration

**Exception Handling Configuration**:
```properties
# Error Handling Configuration
app.error-handling.max-context-size=10
app.error-handling.max-remediation-steps=5
app.error-handling.enable-stack-traces=false
app.error-handling.documentation-base-url=https://docs.example.com/errors

# Circuit Breaker Configuration
mp.fault-tolerance.circuitbreaker.successThreshold=3
mp.fault-tolerance.circuitbreaker.requestVolumeThreshold=4
mp.fault-tolerance.circuitbreaker.failureRatio=0.75
mp.fault-tolerance.circuitbreaker.delay=5000

# Retry Configuration  
mp.fault-tolerance.retry.maxRetries=3
mp.fault-tolerance.retry.delay=1000
mp.fault-tolerance.retry.maxDuration=10000

# Timeout Configuration
mp.fault-tolerance.timeout.value=5000

# OpenTelemetry Configuration
quarkus.opentelemetry.tracer.exporter.otlp.endpoint=http://jaeger:14250
quarkus.opentelemetry.tracer.exporter.otlp.headers=Authorization=Bearer ${TRACING_TOKEN:}

# Logging Configuration
quarkus.log.level=INFO
quarkus.log.console.json=true
quarkus.log.console.json.pretty-print=false
```

**Validation Commands**:
```bash
# Test exception handling patterns
./mvnw test -Dtest=ExceptionHandlingTest

# Verify circuit breaker functionality
./mvnw test -Dtest=CircuitBreakerTest

# Test error context propagation
./mvnw test -Dtest=ErrorContextPropagationTest

# Verify OpenTelemetry integration
curl -H "traceparent: 00-12345678901234567890123456789012-1234567890123456-01" \
     "http://localhost:8080/api/v1/users/invalid-id"

# Check structured logging output
./mvnw quarkus:dev -Dquarkus.log.console.json=true

# Validate native compilation compatibility
./mvnw package -Pnative -DskipTests
```
# Additional Recommendations:

- It is RECOMMENDED to implement an LRU cache for frequently accessed error templates to reduce object creation (https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashMap.html — override `removeEldestEntry`)
- It is RECOMMENDED to process non-critical error logging asynchronously using Mutiny `emitOn` to minimize request-path latency (https://smallrye.io/smallrye-mutiny/latest/guides/emit-on-vs-run-subscription-on/)
- It is RECOMMENDED to implement automated error-pattern alerting via Prometheus `error_total` counter with Grafana alert rules (https://quarkus.io/guides/micrometer)
- Circuit breaker thresholds **SHOULD** be tuned per-service using MicroProfile Config overrides (https://download.eclipse.org/microprofile/microprofile-fault-tolerance-4.1/microprofile-fault-tolerance-spec-4.1.html#_configuration)
# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Global Exception Mapper | Code search | ExceptionMapper implemented |
| Reactive Error Handling | Code inspection | Uni/Multi onFailure handlers |
| Error Response DTO | Code inspection | Standardized error response class |
| Exception Hierarchy | Code inspection | Domain-specific exceptions defined |
| Error Logging | Exception handler code | Errors logged with context |
| No Stack Trace Leaks | API response test | Production hides internals |
| Unit Tests | `./mvnw test -Dtest=*Exception*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Exception Handling (Reactive)..."

FAIL=0

# CRITICAL: Check for prohibited raw RuntimeException throws
echo "Checking for prohibited RuntimeException usage..."
RAW_RUNTIME=$(grep -rn 'throw new RuntimeException' */src/main/java 2>/dev/null | wc -l)
if [ "$RAW_RUNTIME" -eq 0 ]; then
  echo "✅ No raw RuntimeException throws found"
else
  echo "❌ VIOLATION: Found $RAW_RUNTIME raw RuntimeException throws"
  grep -rn 'throw new RuntimeException' */src/main/java 2>/dev/null | head -10
  FAIL=1
fi

# Check for generic Exception throws (also prohibited)
GENERIC_EXCEPTION=$(grep -rn 'throw new Exception(' */src/main/java 2>/dev/null | wc -l)
if [ "$GENERIC_EXCEPTION" -eq 0 ]; then
  echo "✅ No generic Exception throws found"
else
  echo "❌ VIOLATION: Found $GENERIC_EXCEPTION generic Exception throws"
  grep -rn 'throw new Exception(' */src/main/java 2>/dev/null | head -5
  FAIL=1
fi

# Check for ExceptionMapper implementation
if grep -rq "ExceptionMapper\|@Provider" */src/main/java 2>/dev/null; then
  echo "✅ ExceptionMapper implementation found"
else
  echo "❌ No ExceptionMapper found"
  FAIL=1
fi

# Check for reactive error handling patterns
if grep -rq "onFailure\|onError\|recover" */src/main/java 2>/dev/null; then
  echo "✅ Reactive error handling patterns found"
else
  echo "❌ No reactive error handling patterns"
  FAIL=1
fi

# Check for custom exception classes
CUSTOM_EXCEPTIONS=$(find . -name "*Exception.java" -path "*/src/main/*" 2>/dev/null | wc -l)
if [ "$CUSTOM_EXCEPTIONS" -gt 0 ]; then
  echo "✅ Custom exception classes found: $CUSTOM_EXCEPTIONS"
else
  echo "❌ No custom exception classes"
  FAIL=1
fi

# Check for error response DTO
if grep -rq "ErrorResponse\|ProblemDetail\|ErrorDto" */src/main/java 2>/dev/null; then
  echo "✅ Error response DTO found"
else
  echo "❌ No standardized error response DTO"
  FAIL=1
fi

# Check that exceptions don't expose internal details in mappers
if grep -rn "getStackTrace" */src/main/java/**/mapper 2>/dev/null | grep -v test | grep -q .; then
  echo "❌ Potential stack trace exposure in mappers"
  FAIL=1
else
  echo "✅ No obvious stack trace leaks"
fi

# Run exception-related tests
./mvnw test -Dtest="*Exception*" -q 2>/dev/null && echo "✅ Exception tests pass" || { echo "❌ Exception tests failed or not found"; FAIL=1; }

# Build verification
./mvnw clean compile -q && echo "✅ Build successful" || { echo "❌ Build failed"; FAIL=1; }

if [ $FAIL -ne 0 ]; then
  echo "❌ Exception Handling (Reactive) validation FAILED"
  exit 1
fi
echo "✅ All Exception Handling (Reactive) criteria met"
```
