# Issue: REST API Error Handling Strategy

Quarkus microservices require standardized REST API error handling for API contract stability, operational resilience, and reactive programming integration.

# Decision:

Implement RFC 9457-compliant REST API error handling with structured Problem Details format using Quarkus reactive exception mapping. All REST endpoints return consistent error responses with machine-readable error codes, proper HTTP status codes, and integrate with OpenTelemetry tracing and structured logging.

**Required Error Handling Components**:
- RFC 9457 Problem Details format for all error responses
- Reactive exception mapper with non-blocking error transformation
- Machine-readable error codes with HTTP status code mapping
- Error context propagation across reactive streams
- OpenTelemetry span integration for distributed tracing
- Structured logging with consistent error event format
- Hexagonal architecture boundary exception translation
- GraalVM native compilation compatibility


# Constraints:

- Error responses **MUST** comply with RFC 9457 (Problem Details for HTTP APIs) specification
- HTTP status codes **MUST** follow semantic meanings defined in RFC 7231
- Content-Type **MUST** be `application/problem+json` for structured error responses
- Problem Details fields **MUST NOT** contain sensitive information
- Existing API contracts **MUST NOT** be broken for backward compatibility
- Error response format changes **SHALL** be additive only
- Error response generation **MUST NOT** introduce blocking operations in reactive streams
- Exception mapping overhead **MUST NOT** exceed 2ms per error response generation
- Solution **MUST** be compatible with Quarkus native compilation (GraalVM)
- Error handling **MUST** integrate with service mesh observability tools


# Alternatives:

### 1. Plain Text or Ad-hoc JSON Error Responses

Returning unstructured error strings or application-specific JSON formats requires every API client to implement custom parsing per service. RFC 9457 (https://www.rfc-editor.org/rfc/rfc9457) defines a standard `application/problem+json` media type that HTTP client libraries and API gateways can parse without service-specific logic. Without this standard, API gateway error aggregation tools such as Kong and Envoy cannot automatically classify error responses by type and status (https://gateway-api.sigs.k8s.io/api-types/httproute/#filters).

### 2. Jakarta REST `WebApplicationException` Only

Relying solely on JAX-RS `WebApplicationException` subclasses (`NotFoundException`, `BadRequestException`) provides HTTP status codes but no machine-readable error codes, correlation IDs, or remediation hints. The Quarkus RESTEasy Reactive exception mapping documentation (https://quarkus.io/guides/resteasy-reactive#exception-mapping) demonstrates that custom `ExceptionMapper<T>` implementations are required to control response body format. Using only `WebApplicationException` produces plain text error bodies that cannot carry the extension fields (`errorCode`, `traceId`, `retryable`) needed for distributed tracing correlation (ADR-21) and structured logging (ADR-16).


# Rationale:

RFC 9457 (https://www.rfc-editor.org/rfc/rfc9457) defines the `application/problem+json` media type with required fields (`type`, `title`, `status`, `detail`, `instance`) and support for extension members. This enables API clients to use a single error parser across all services, as the format is registered with IANA (https://www.iana.org/assignments/media-types/application/problem+json).

Quarkus RESTEasy Reactive supports custom `ExceptionMapper<T>` implementations that intercept exceptions and produce structured responses without blocking the I/O thread (https://quarkus.io/guides/resteasy-reactive#exception-mapping). The `@Provider` annotation registers the mapper globally, ensuring all unhandled exceptions produce RFC 9457 responses.

Extension fields (`errorCode`, `correlationId`, `traceId`, `spanId`) enable correlation between error responses, structured log entries (ADR-16), distributed traces (ADR-21), and audit events (ADR-19). The `@RegisterForReflection` annotation ensures ProblemDetail serialization works under GraalVM native compilation (https://quarkus.io/guides/writing-native-applications-tips#registering-for-reflection).

# Implementation Guidelines:

## Naming Conventions:
- Exception mappers: `[ExceptionType]ExceptionMapper`
- Problem detail builders: `[ErrorType]ProblemDetailBuilder`
- Error context managers: `[Layer]ErrorContextManager`
- HTTP status mappers: `[Domain]HttpStatusMapper`
- Error response DTOs: `[ErrorType]ProblemDetail`
- Error logging services: `[Layer]ErrorLogger`

## 1. RFC 9457 Problem Details Implementation

**Complete Problem Detail Model**:
```java
@JsonInclude(JsonInclude.Include.NON_NULL)
@RegisterForReflection // GraalVM native compilation support
public class ProblemDetail {
    
    @JsonProperty("type")
    private URI type;
    
    @JsonProperty("title") 
    private String title;
    
    @JsonProperty("status")
    private Integer status;
    
    @JsonProperty("detail")
    private String detail;
    
    @JsonProperty("instance")
    private String instance;
    
    // RFC 9457 extension fields for enhanced error context
    @JsonProperty("errorCode")
    private String errorCode;
    
    @JsonProperty("correlationId")
    private String correlationId;
    
    @JsonProperty("timestamp")
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private Instant timestamp;
    
    @JsonProperty("applicationVersion")
    private String applicationVersion;
    
    @JsonProperty("retryable")
    private Boolean retryable;
    
    @JsonProperty("remediationSteps")
    private List<String> remediationSteps;
    
    @JsonProperty("traceId")
    private String traceId;
    
    @JsonProperty("spanId")  
    private String spanId;
    
    // Builder pattern for fluent construction
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private final ProblemDetail problem = new ProblemDetail();
        
        public Builder type(URI type) {
            problem.type = type;
            return this;
        }
        
        public Builder errorCode(String errorCode) {
            problem.errorCode = errorCode;
            problem.type = URI.create(String.format("https://api.copilot-quarkus.com/problems/%s", errorCode));
            return this;
        }
        
        public Builder retryable(boolean retryable) {
            problem.retryable = retryable;
            return this;
        }
        
        public Builder traceContext() {
            Span currentSpan = Span.current();
            SpanContext spanContext = currentSpan.getSpanContext();
            problem.traceId = spanContext.getTraceId();
            problem.spanId = spanContext.getSpanId();
            return this;
        }
        
        public ProblemDetail build() {
            if (problem.timestamp == null) {
                problem.timestamp = Instant.now();
            }
            return problem;
        }
    }
}
```

**HTTP Status Code Mapping Service**:
```java
@ApplicationScoped
public class HttpStatusMapper {
    
    private static final Map<String, Integer> DOMAIN_ERROR_STATUS_MAP = Map.of(
        "DOM-001", 409, // Conflict - Email already exists
        "DOM-002", 404, // Not Found - Entity not found
        "DOM-003", 400, // Bad Request - Validation failure
        "DOM-004", 422, // Unprocessable Entity - Business state error
        "DOM-005", 403  // Forbidden - Authorization failure
    );
    
    private static final Map<String, Integer> INFRASTRUCTURE_ERROR_STATUS_MAP = Map.of(
        "INF-001", 502, // Bad Gateway - External service failure
        "INF-002", 503, // Service Unavailable - Database connection failure
        "INF-003", 504, // Gateway Timeout - External service timeout
        "INF-004", 500, // Internal Server Error - Configuration issue
        "INF-005", 503  // Service Unavailable - Circuit breaker open
    );
    
    private static final Map<String, String> ERROR_TITLE_MAP = Map.of(
        "DOM-001", "Business Rule Violation",
        "DOM-002", "Resource Not Found", 
        "DOM-003", "Validation Error",
        "INF-001", "External Service Error",
        "INF-002", "Database Connection Error"
    );
    
    public int mapDomainException(DomainException exception) {
        return DOMAIN_ERROR_STATUS_MAP.getOrDefault(exception.getErrorCode(), 400);
    }
    
    public int mapInfrastructureException(InfrastructureException exception) {
        return INFRASTRUCTURE_ERROR_STATUS_MAP.getOrDefault(exception.getErrorCode(), 500);
    }
    
    public String mapErrorTitle(String errorCode) {
        return ERROR_TITLE_MAP.getOrDefault(errorCode, "Application Error");
    }
    
    public boolean isRetryable(String errorCode) {
        return errorCode.startsWith("INF-002") || // Database connection issues
               errorCode.startsWith("INF-003") || // Timeout issues
               errorCode.startsWith("INF-005");   // Circuit breaker temporary failures
    }
}
```

## 2. Reactive Exception Mapper Implementation

**Core Exception Mapper with Reactive Integration**:
```java
@Provider
@ApplicationScoped
public class ReactiveExceptionMapper implements ExceptionMapper<Throwable> {
    
    @Inject
    ErrorContextManager errorContextManager;
    
    @Inject
    HttpStatusMapper httpStatusMapper;
    
    @Inject
    StructuredErrorLogger errorLogger;
    
    @Inject
    @ConfigProperty(name = "quarkus.application.version", defaultValue = "unknown")
    String applicationVersion;
    
    @Override
    public Response toResponse(Throwable exception) {
        ErrorContext context = errorContextManager.getCurrentContext();
        
        return switch (exception) {
            case DomainException de -> createDomainProblemResponse(de, context);
            case InfrastructureException ie -> createInfrastructureProblemResponse(ie, context);
            case ValidationException ve -> createValidationProblemResponse(ve, context);
            case WebApplicationException wae -> createWebApplicationProblemResponse(wae, context);
            default -> createGenericProblemResponse(exception, context);
        };
    }
    
    private Response createDomainProblemResponse(DomainException exception, ErrorContext context) {
        ProblemDetail problem = ProblemDetail.builder()
            .errorCode(exception.getErrorCode())
            .title(httpStatusMapper.mapErrorTitle(exception.getErrorCode()))
            .status(httpStatusMapper.mapDomainException(exception))
            .detail(exception.getMessage())
            .instance(context.getCorrelationId())
            .correlationId(context.getCorrelationId())
            .applicationVersion(applicationVersion)
            .retryable(false) // Domain errors typically not retryable
            .traceContext()
            .build();
            
        // Log error for observability
        errorLogger.logDomainError(problem, exception);
        
        return Response.status(problem.getStatus())
            .entity(problem)
            .type("application/problem+json")
            .header("X-Correlation-Id", context.getCorrelationId())
            .header("X-Error-Code", exception.getErrorCode())
            .build();
    }
    
    private Response createInfrastructureProblemResponse(InfrastructureException exception, ErrorContext context) {
        ProblemDetail problem = ProblemDetail.builder()
            .errorCode(exception.getErrorCode())
            .title(httpStatusMapper.mapErrorTitle(exception.getErrorCode()))
            .status(httpStatusMapper.mapInfrastructureException(exception))
            .detail("A technical error occurred. Please try again later.")
            .instance(context.getCorrelationId())
            .correlationId(context.getCorrelationId())
            .applicationVersion(applicationVersion)
            .retryable(httpStatusMapper.isRetryable(exception.getErrorCode()))
            .traceContext()
            .build();
        
        // Log error with full technical details
        errorLogger.logInfrastructureError(problem, exception);
        
        return Response.status(problem.getStatus())
            .entity(problem)
            .type("application/problem+json")
            .header("X-Correlation-Id", context.getCorrelationId())
            .header("X-Error-Code", exception.getErrorCode())
            .header("Retry-After", httpStatusMapper.isRetryable(exception.getErrorCode()) ? "30" : null)
            .build();
    }
}
```

**Error Context Manager for Reactive Streams**:
```java
@ApplicationScoped
public class ErrorContextManager {
    
    private static final String CORRELATION_ID_KEY = "correlationId";
    private static final String OPERATION_KEY = "operation";
    private static final String USER_ID_KEY = "userId";
    
    public ErrorContext getCurrentContext() {
        return ErrorContext.builder()
            .correlationId(getCorrelationId())
            .operationName(getCurrentOperation())
            .userId(getCurrentUserId())
            .timestamp(Instant.now())
            .build();
    }
    
    private String getCorrelationId() {
        // Try context propagation first (reactive safe)
        return Optional.ofNullable(Context.get(CORRELATION_ID_KEY))
            .or(() -> Optional.ofNullable(MDC.get(CORRELATION_ID_KEY))) // Fallback for non-reactive
            .orElse(generateCorrelationId());
    }
    
    private String getCurrentOperation() {
        return Optional.ofNullable(Context.get(OPERATION_KEY))
            .or(() -> Optional.ofNullable(MDC.get(OPERATION_KEY)))
            .orElse("UnknownOperation");
    }
    
    private String getCurrentUserId() {
        return Optional.ofNullable(Context.get(USER_ID_KEY))
            .or(() -> Optional.ofNullable(MDC.get(USER_ID_KEY)))
            .orElse("anonymous");
    }
    
    public <T> Uni<T> withErrorContext(Uni<T> operation, String correlationId, String operationName, String userId) {
        return operation
            .emitOn(Infrastructure.getDefaultExecutor())
            .invoke(() -> {
                Context.put(CORRELATION_ID_KEY, correlationId);
                Context.put(OPERATION_KEY, operationName);
                Context.put(USER_ID_KEY, userId);
                
                // Also set MDC for non-reactive compatibility
                MDC.put(CORRELATION_ID_KEY, correlationId);
                MDC.put(OPERATION_KEY, operationName);
                MDC.put(USER_ID_KEY, userId);
            })
            .onTermination().invoke(() -> {
                // Clean up context
                Context.remove(CORRELATION_ID_KEY);
                Context.remove(OPERATION_KEY);
                Context.remove(USER_ID_KEY);
                MDC.clear();
            });
    }
    
    private String generateCorrelationId() {
        return UUID.randomUUID().toString().replace("-", "").substring(0, 12).toUpperCase();
    }
}
```

## 3. Hexagonal Architecture Integration

**REST Adapter with Exception Translation**:
```java
@Path("/api/v1/users")
@ApplicationScoped
@Tag(name = "Users", description = "User management operations")
@SecurityRequirement(name = "jwt")
public class UserRestAdapter {
    
    @Inject
    UserService userService; // Domain service port
    
    @Inject
    ErrorContextManager errorContextManager;
    
    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @Operation(summary = "Create new user", description = "Creates a new user in the system")
    @APIResponses({
        @APIResponse(responseCode = "201", description = "User created successfully",
                    content = @Content(schema = @Schema(implementation = UserResponse.class))),
        @APIResponse(responseCode = "400", description = "Validation error",
                    content = @Content(schema = @Schema(implementation = ProblemDetail.class))),
        @APIResponse(responseCode = "409", description = "Email already exists",
                    content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
    })
    public Uni<Response> createUser(CreateUserRequest request, @Context SecurityContext securityContext) {
        String correlationId = generateCorrelationId();
        String userId = securityContext.getUserPrincipal().getName();
        
        return errorContextManager.withErrorContext(
            userService.createUser(request.toCommand()),
            correlationId,
            "CreateUser",
            userId
        )
        .onItem().transform(user -> Response.status(201)
            .entity(UserResponse.from(user))
            .header("X-Correlation-Id", correlationId)
            .build())
        .onFailure().recoverWithItem(failure -> {
            // Exception mapper will handle the transformation
            throw (RuntimeException) failure;
        });
    }
    
    @GET
    @Path("/{id}")
    @Produces(MediaType.APPLICATION_JSON)
    @Operation(summary = "Get user by ID")
    @APIResponses({
        @APIResponse(responseCode = "200", description = "User found",
                    content = @Content(schema = @Schema(implementation = UserResponse.class))),
        @APIResponse(responseCode = "404", description = "User not found",
                    content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
    })
    public Uni<Response> getUser(@PathParam("id") String userId, @Context SecurityContext securityContext) {
        String correlationId = getCorrelationId();
        String currentUser = securityContext.getUserPrincipal().getName();
        
        return errorContextManager.withErrorContext(
            userService.findById(UserId.of(userId)),
            correlationId,
            "GetUser",
            currentUser
        )
        .onItem().transform(user -> Response.ok(UserResponse.from(user))
            .header("X-Correlation-Id", correlationId)
            .build());
    }
    
    private String generateCorrelationId() {
        return UUID.randomUUID().toString().replace("-", "").substring(0, 12);
    }
    
    private String getCorrelationId() {
        return Optional.ofNullable(MDC.get("correlationId"))
                .orElse(generateCorrelationId());
    }
}
```

## 4. OpenTelemetry and Structured Logging Integration

**Structured Error Logger**:
```java
@ApplicationScoped
public class StructuredErrorLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(StructuredErrorLogger.class);
    
    @Inject
    @ConfigProperty(name = "quarkus.application.version", defaultValue = "unknown")
    String applicationVersion;
    
    public void logDomainError(ProblemDetail problem, DomainException exception) {
        Map<String, Object> logEntry = createBaseLogEntry(problem);
        logEntry.put("error_category", "DOMAIN");
        logEntry.put("business_context", exception.getErrorContext().getBusinessContext());
        logEntry.put("remediation_steps", extractRemediationSteps(exception.getErrorCode()));
        
        logger.error("Domain error occurred: {}", logEntry, exception);
    }
    
    public void logInfrastructureError(ProblemDetail problem, InfrastructureException exception) {
        Map<String, Object> logEntry = createBaseLogEntry(problem);
        logEntry.put("error_category", "INFRASTRUCTURE");
        logEntry.put("retryable", problem.getRetryable());
        logEntry.put("technical_context", exception.getErrorContext().getTechnicalContext());
        
        logger.error("Infrastructure error occurred: {}", logEntry, exception);
    }
    
    private Map<String, Object> createBaseLogEntry(ProblemDetail problem) {
        return Map.of(
            "event_type", "REST_API_ERROR",
            "error_code", problem.getErrorCode(),
            "correlation_id", problem.getCorrelationId(),
            "http_status", problem.getStatus(),
            "timestamp", problem.getTimestamp(),
            "application_version", applicationVersion,
            "trace_id", problem.getTraceId(),
            "span_id", problem.getSpanId()
        );
    }
    
    private List<String> extractRemediationSteps(String errorCode) {
        return switch (errorCode) {
            case "DOM-001" -> List.of("Use a different email address", "Contact support if this is unexpected");
            case "DOM-002" -> List.of("Verify the resource ID", "Check if the resource was recently deleted");
            case "INF-001" -> List.of("Try again in a few moments", "Contact support if the issue persists");
            default -> List.of("Contact technical support for assistance");
        };
    }
}
```

**OpenTelemetry Integration**:
```java
@ApplicationScoped
public class TracingErrorHandler {
    
    @Traced(operationName = "error-handling")
    public void enhanceSpanWithErrorContext(Throwable error, ProblemDetail problemDetail) {
        Span currentSpan = Span.current();
        
        // Set error status on span
        currentSpan.setStatus(StatusCode.ERROR, error.getMessage());
        currentSpan.recordException(error);
        
        // Add structured attributes
        currentSpan.setAttributes(Attributes.of(
            AttributeKey.stringKey("error.type"), error.getClass().getSimpleName(),
            AttributeKey.stringKey("error.code"), problemDetail.getErrorCode(),
            AttributeKey.longKey("http.status_code"), Long.valueOf(problemDetail.getStatus())),
            AttributeKey.stringKey("correlation.id"), problemDetail.getCorrelationId(),
            AttributeKey.booleanKey("error.retryable"), problemDetail.getRetryable() != null ? problemDetail.getRetryable() : false
        );
        
        // Create error event with additional context
        currentSpan.addEvent("rest_api_error", Attributes.of(
            AttributeKey.stringKey("error.message"), error.getMessage(),
            AttributeKey.longKey("error.timestamp"), problemDetail.getTimestamp().toEpochMilli(),
            AttributeKey.stringKey("problem.type"), problemDetail.getType().toString()
        ));
    }
}
```

## 5. Application Configuration

**Error Handling Configuration Properties**:
```properties
# REST API Error Handling Configuration
app.error-handling.include-stack-traces=false
app.error-handling.expose-technical-details=false
app.error-handling.problem-detail-base-url=https://api.copilot-quarkus.com/problems
app.error-handling.default-retry-after-seconds=30

# HTTP Response Configuration
quarkus.http.header."X-Content-Type-Options".value=nosniff
quarkus.http.header."X-Frame-Options".value=DENY
quarkus.http.header."Cache-Control".value=no-cache, no-store, max-age=0

# OpenTelemetry Integration
quarkus.opentelemetry.tracer.exporter.otlp.endpoint=http://jaeger:14250
quarkus.opentelemetry.tracer.exporter.otlp.headers=Authorization=Bearer ${TRACING_TOKEN:}

# Structured Logging
quarkus.log.level=INFO
quarkus.log.console.json=true
quarkus.log.console.json.pretty-print=false
```

**Validation Commands**:
```bash
# Test exception mapping with different error types
curl -X POST "http://localhost:8080/api/v1/users" \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $TOKEN" \
     -d '{"email":"existing@example.com","firstName":"Test","lastName":"User"}'

# Verify RFC 9457 compliance in response
curl -X GET "http://localhost:8080/api/v1/users/nonexistent" \
     -H "Accept: application/problem+json" \
     -H "Authorization: Bearer $TOKEN" | jq .

# Test correlation ID propagation
curl -X GET "http://localhost:8080/api/v1/users/invalid-id" \
     -H "X-Correlation-ID: test-123" \
     -H "Authorization: Bearer $TOKEN"

# Verify error response headers
curl -I "http://localhost:8080/api/v1/users/invalid" \
     -H "Authorization: Bearer $TOKEN"

# Test exception mapping in tests
./mvnw test -Dtest=RestExceptionMappingTest

# Verify OpenTelemetry trace integration
./mvnw test -Dtest=TracingErrorHandlerTest

# Validate native compilation compatibility
./mvnw package -Pnative -DskipTests
```

# Additional Recommendations:

- **RECOMMENDED**: Cache frequently accessed `ProblemDetail` type URIs using a static map to reduce object allocation — see RFC 9457 `type` field specification (https://www.rfc-editor.org/rfc/rfc9457#section-3.1.1).
- **RECOMMENDED**: Configure Prometheus counters for error rates grouped by `errorCode` and HTTP status — see Quarkus Micrometer guide (https://quarkus.io/guides/telemetry-micrometer).
- **RECOMMENDED**: Maintain a centralized error code registry with remediation steps — see RFC 9457 extension members (https://www.rfc-editor.org/rfc/rfc9457#section-3.2).
- **RECOMMENDED**: Document all error responses in OpenAPI using `@APIResponse` annotations — see Quarkus OpenAPI guide (https://quarkus.io/guides/openapi-swaggerui).
- **RECOMMENDED**: Use Quarkus `@ServerExceptionMapper` for type-safe reactive exception mapping — see RESTEasy Reactive exception handling (https://quarkus.io/guides/resteasy-reactive#exception-mapping).

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| RFC 9457 Compliance | API error response | application/problem+json content-type |
| Problem Detail Fields | Error response body | type, title, status, detail, instance |
| Validation Errors | 400 response | Field-level errors with path |
| Not Found Errors | 404 response | Proper problem detail |
| Server Errors | 500 response | No internal details exposed |
| Error Documentation | OpenAPI spec | Error responses documented |
| Integration Tests | `./mvnw test -Dtest=*Error*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating REST API Error Handling..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
fi

# Check for ProblemDetail or RFC 9457 implementation
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "ProblemDetail\|application/problem" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ RFC 9457 Problem Detail implementation found"
else
  echo "❌ No RFC 9457 ProblemDetail implementation found"
  FAIL=1
fi

# Check for problem+json content-type
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "application/problem" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Problem JSON content-type configured"
else
  echo "❌ Problem JSON content-type (application/problem+json) not found"
  FAIL=1
fi

# Check for ExceptionMapper implementation
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "ExceptionMapper\|@ServerExceptionMapper" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Exception mapper implementation found"
else
  echo "❌ No ExceptionMapper or @ServerExceptionMapper found"
  FAIL=1
fi

# Check for constraint violation handling
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "ConstraintViolation\|ValidationException" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Validation error handling found"
else
  echo "❌ No ConstraintViolation/ValidationException handling found"
  FAIL=1
fi

# Check for OpenAPI error documentation
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "@APIResponse" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Error responses documented in OpenAPI"
else
  echo "❌ No @APIResponse annotations for error documentation"
  FAIL=1
fi

# Check for RegisterForReflection on ProblemDetail (GraalVM)
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "@RegisterForReflection" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ @RegisterForReflection annotation found (GraalVM support)"
else
  echo "❌ No @RegisterForReflection found for native compilation"
  FAIL=1
fi

# Run error handling tests
if ./mvnw test -Dtest="*Error*,*Exception*" -q 2>/dev/null; then
  echo "✅ Error handling tests pass"
else
  echo "❌ Error handling tests failed or not found"
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
  echo "❌ REST API Error Handling validation failed"
  exit 1
fi
echo "✅ All REST API Error Handling criteria met"
```
