# Issue: Distributed Tracing Implementation for Microservices

Microservices architectures without distributed tracing lack end-to-end visibility for request flows spanning multiple services. Traditional logging fails to correlate events across service boundaries, making root cause analysis difficult and increasing Mean Time To Resolution (MTTR) for cross-service issues.

# Decision:

Implement OpenTelemetry-based distributed tracing with Jaeger backend for comprehensive trace collection and analysis. All HTTP endpoints, database operations, and critical business operations **MUST** be automatically instrumented with minimal performance overhead.

**Required Components**:
- **OpenTelemetry SDK**: Quarkus OpenTelemetry extension for automatic and manual instrumentation
- **Jaeger Backend**: All-in-one Jaeger deployment for trace collection, storage, and visualization
- **OTLP Exporter**: gRPC-based trace export to Jaeger collector
- **Context Propagation**: Automatic trace context propagation across reactive Mutiny streams
- **Custom Instrumentation**: Manual span creation for business-critical operations

# Constraints:

- All tracing components **MUST** work with GraalVM native compilation
- Tracing infrastructure **MUST NOT** introduce blocking operations in reactive streams
- Memory overhead **MUST NOT** exceed 5% of total application memory usage
- Runtime overhead **MUST NOT** add more than 2ms latency to simple operations
- Production environments **MUST** use probabilistic sampling to control overhead (1-10%)
- **MUST** support automatic context propagation across reactive Mutiny streams
- Trace data **MUST NOT** contain sensitive information (credentials, PII, tokens)

# Alternatives:

### 1. Zipkin

Zipkin supports trace collection via HTTP and Kafka but lacks built-in service dependency graph generation — the dependency endpoint requires a separate Spark job for aggregation (https://zipkin.io/pages/architecture.html). Zipkin's query API does not support tag-based trace search filtering, which Jaeger provides natively through its gRPC query service (https://www.jaegertracing.io/docs/latest/apis/#grpc-query-service). Additionally, Zipkin does not support trace comparison, a feature Jaeger includes for A/B debugging of latency regressions (https://www.jaegertracing.io/docs/latest/frontend-ui/#trace-comparison).

### 2. AWS X-Ray / Google Cloud Trace

Cloud-provider tracing services require vendor-specific SDKs and data formats. AWS X-Ray uses a proprietary segment document format incompatible with OpenTelemetry's OTLP protocol without an intermediary collector (https://docs.aws.amazon.com/xray/latest/devguide/xray-concepts.html). This creates cloud vendor lock-in and prevents local development tracing without cloud connectivity, contradicting the Docker Compose-based development environment defined in ADR-07.

# Rationale:

Quarkus provides the `quarkus-opentelemetry` extension which auto-instruments RESTEasy Reactive endpoints, REST client calls, and gRPC services without application code changes (https://quarkus.io/guides/opentelemetry). The extension integrates with SmallRye Context Propagation to maintain trace context across Mutiny reactive pipelines, including `Uni` and `Multi` chains (https://quarkus.io/guides/opentelemetry#reactive).

Jaeger receives traces via the OTLP gRPC protocol (port 4317) and OTLP HTTP protocol (port 4318), both supported by the Quarkus OpenTelemetry exporter (https://www.jaegertracing.io/docs/latest/apis/#opentelemetry-protocol-stable). Jaeger's all-in-one deployment includes the collector, query service, and UI in a single container, with in-memory or Badger storage for development (https://www.jaegertracing.io/docs/latest/deployment/#all-in-one).

Production sampling is configured via `quarkus.otel.traces.sampler=parentbased_traceidratio`, which respects upstream sampling decisions while applying a configurable local ratio, as documented in the OpenTelemetry SDK specification (https://opentelemetry.io/docs/specs/otel/trace/sdk/#parentbased).

# Implementation Guidelines:

## Naming Conventions:
- Custom spans: `[ClassName].[methodName]` (e.g., `UserService.createUser`)
- Span attributes: lowercase with dots (e.g., `user.email`, `http.route`)
- Span events: PascalCase (e.g., `ValidationCompleted`, `DatabaseQueryExecuted`)

## 1. Maven Dependencies

Add the Quarkus OpenTelemetry extension:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-opentelemetry</artifactId>
</dependency>
```

## 2. Application Configuration

```properties
# =============================================================================
# DISTRIBUTED TRACING CONFIGURATION
# =============================================================================

# OpenTelemetry Core Settings
quarkus.otel.enabled=true
quarkus.otel.service.name=${quarkus.application.name}

# Sampling Configuration
# Development: 100% sampling for full visibility
quarkus.otel.traces.sampler=traceidratio
quarkus.otel.traces.sampler.arg=1.0

# Production: Lower sampling rate (10%) to reduce overhead
%prod.quarkus.otel.traces.sampler.arg=0.1

# OTLP Exporter Configuration
quarkus.otel.exporter.otlp.traces.endpoint=${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4317}
quarkus.otel.exporter.otlp.traces.protocol=grpc

# Resource Attributes (appear on all spans)
quarkus.otel.resource.attributes=service.namespace=copilot-demo,service.version=${quarkus.application.version},deployment.environment=${ENVIRONMENT:development}

# Propagation (W3C Trace Context is default)
quarkus.otel.propagators=tracecontext,baggage

# Span Processor Configuration
quarkus.otel.bsp.schedule.delay=5000
quarkus.otel.bsp.max.queue.size=2048
quarkus.otel.bsp.max.export.batch.size=512
quarkus.otel.bsp.export.timeout=30000
```

## 3. Docker Compose Integration

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    container_name: jaeger
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317:4317"    # OTLP gRPC receiver
      - "4318:4318"    # OTLP HTTP receiver
      - "14250:14250"  # Model proto (internal)
      - "14268:14268"  # Jaeger thrift HTTP
      - "6831:6831/udp" # Jaeger compact thrift (legacy)
    environment:
      COLLECTOR_OTLP_ENABLED: true
      SPAN_STORAGE_TYPE: memory  # Use 'badger' or 'elasticsearch' for persistence
      QUERY_MAX_CLOCK_SKEW_ADJUSTMENT: 1s
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:16686"]
      interval: 10s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1'
```

## 4. Service Layer Instrumentation

### Automatic Instrumentation with @WithSpan

```java
package com.copilot.quarkus.application.user.service;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.instrumentation.annotations.SpanAttribute;
import io.opentelemetry.instrumentation.annotations.WithSpan;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

/**
 * User application service with distributed tracing instrumentation.
 */
@ApplicationScoped
public class UserApplicationService {

    @Inject
    UserRepositoryPort userRepository;

    /**
     * Create a new user with comprehensive tracing.
     * 
     * @WithSpan creates a new span for this method invocation.
     * @SpanAttribute captures method parameters as span attributes.
     */
    @WithSpan(value = "UserService.createUser", kind = SpanKind.INTERNAL)
    public Uni<User> createUser(
            @SpanAttribute("user.email") String email,
            @SpanAttribute("user.firstName") String firstName,
            @SpanAttribute("user.lastName") String lastName) {
        
        // Add additional context to the current span
        Span.current().setAttribute("user.operation", "CREATE");
        
        User user = new User(email, firstName, lastName);
        
        return userRepository.create(user)
            .invoke(created -> {
                // Record success event
                Span.current().addEvent("UserCreated", 
                    Attributes.of(AttributeKey.stringKey("user.id"), created.getId().toString()));
            })
            .onFailure().invoke(error -> {
                // Record error in span
                Span.current().recordException(error);
                Span.current().setStatus(StatusCode.ERROR, error.getMessage());
            });
    }

    @WithSpan("UserService.findById")
    public Uni<User> findById(@SpanAttribute("user.id") Long userId) {
        return userRepository.findById(userId)
            .onItem().ifNull().invoke(() -> 
                Span.current().setStatus(StatusCode.ERROR, "User not found"));
    }

    @WithSpan("UserService.findAll")
    public Multi<User> findAll() {
        Span.current().setAttribute("user.operation", "LIST");
        return userRepository.findAll();
    }
}
```

### Manual Span Creation with Tracer

```java
package com.copilot.quarkus.infrastructure.adapter;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanBuilder;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

/**
 * Database adapter with manual span creation for fine-grained tracing.
 */
@ApplicationScoped
public class ClickHouseUserAdapter implements UserRepositoryPort {

    @Inject
    Tracer tracer;

    @Inject
    ClickHouseClient clickHouseClient;

    @Override
    public Uni<User> findById(Long id) {
        // Create a new span manually
        SpanBuilder spanBuilder = tracer.spanBuilder("ClickHouse.findUserById")
            .setSpanKind(SpanKind.CLIENT)
            .setAttribute("db.system", "clickhouse")
            .setAttribute("db.operation", "SELECT")
            .setAttribute("db.table", "users")
            .setAttribute("user.id", id);

        return Uni.createFrom().deferred(() -> {
            Span span = spanBuilder.startSpan();
            
            return Uni.createFrom().context(ctx -> ctx.put("span", span))
                .chain(() -> executeQuery(id))
                .invoke(user -> {
                    span.setStatus(StatusCode.OK);
                    if (user != null) {
                        span.setAttribute("user.found", true);
                    }
                })
                .onFailure().invoke(error -> {
                    span.recordException(error);
                    span.setStatus(StatusCode.ERROR, error.getMessage());
                })
                .eventually(() -> {
                    span.end();
                    return Uni.createFrom().voidItem();
                });
        });
    }

    private Uni<User> executeQuery(Long id) {
        String sql = "SELECT * FROM users WHERE id = ?";
        // Database query implementation
        return clickHouseClient.query(sql, id);
    }
}
```

## 5. Controller Layer Instrumentation

```java
package com.copilot.quarkus.infrastructure.web.controller;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.instrumentation.annotations.WithSpan;
import io.smallrye.mutiny.Uni;
import jakarta.inject.Inject;
import jakarta.validation.Valid;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

/**
 * User REST controller with distributed tracing.
 */
@Path("/api/v1/users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserController {

    @Inject
    UserApplicationService userService;

    @POST
    @WithSpan("UserController.createUser")
    public Uni<Response> createUser(@Valid CreateUserRequest request) {
        // Add route information to span
        Span.current().setAttribute("http.route", "/api/v1/users");
        Span.current().setAttribute("request.email", request.getEmail());
        
        return userService.createUser(
                request.getEmail(),
                request.getFirstName(),
                request.getLastName())
            .map(user -> Response.status(Response.Status.CREATED)
                .entity(UserResponse.from(user))
                .build())
            .onFailure().recoverWithItem(error -> {
                Span.current().recordException(error);
                return Response.status(Response.Status.BAD_REQUEST)
                    .entity(ProblemDetail.from(error))
                    .build();
            });
    }

    @GET
    @Path("/{id}")
    @WithSpan("UserController.getUserById")
    public Uni<UserResponse> getUserById(@PathParam("id") Long id) {
        Span.current().setAttribute("http.route", "/api/v1/users/{id}");
        Span.current().setAttribute("user.id", id);
        
        return userService.findById(id)
            .map(UserResponse::from);
    }
}
```

## 6. Context Propagation in Reactive Streams

```java
package com.copilot.quarkus.infrastructure.tracing;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

/**
 * Utility for propagating trace context in reactive streams.
 */
@ApplicationScoped
public class TracingContextPropagator {

    @Inject
    Tracer tracer;

    /**
     * Execute a reactive operation within a traced context.
     */
    public <T> Uni<T> withTracing(String spanName, java.util.function.Supplier<Uni<T>> operation) {
        return Uni.createFrom().deferred(() -> {
            Span span = tracer.spanBuilder(spanName).startSpan();
            Context parentContext = Context.current().with(span);
            
            try (Scope scope = parentContext.makeCurrent()) {
                return operation.get()
                    .invoke(() -> span.setStatus(StatusCode.OK))
                    .onFailure().invoke(error -> {
                        span.recordException(error);
                        span.setStatus(StatusCode.ERROR, error.getMessage());
                    })
                    .eventually(() -> {
                        span.end();
                        return Uni.createFrom().voidItem();
                    });
            }
        });
    }

    /**
     * Add an event to the current span.
     */
    public void addEvent(String eventName, Map<String, String> attributes) {
        AttributesBuilder builder = Attributes.builder();
        attributes.forEach((k, v) -> builder.put(AttributeKey.stringKey(k), v));
        Span.current().addEvent(eventName, builder.build());
    }

    /**
     * Get current trace context for propagation to external services.
     */
    public Map<String, String> getTraceHeaders() {
        Map<String, String> headers = new HashMap<>();
        Span span = Span.current();
        if (span.getSpanContext().isValid()) {
            headers.put("traceparent", String.format("00-%s-%s-01",
                span.getSpanContext().getTraceId(),
                span.getSpanContext().getSpanId()));
        }
        return headers;
    }
}
```

## 7. Log Correlation Integration

Ensure trace IDs appear in structured logs (see ADR-16):

```java
package com.copilot.quarkus.infrastructure.logging;

import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanContext;
import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ContainerRequestFilter;
import jakarta.ws.rs.container.ContainerResponseFilter;
import jakarta.ws.rs.ext.Provider;
import org.jboss.logmanager.MDC;

/**
 * Filter to propagate trace context to MDC for structured logging.
 */
@Provider
public class TraceContextLoggingFilter implements ContainerRequestFilter, ContainerResponseFilter {

    @Override
    public void filter(ContainerRequestContext requestContext) {
        SpanContext spanContext = Span.current().getSpanContext();
        if (spanContext.isValid()) {
            MDC.put("traceId", spanContext.getTraceId());
            MDC.put("spanId", spanContext.getSpanId());
        }
    }

    @Override
    public void filter(ContainerRequestContext request, ContainerResponseContext response) {
        // Add trace ID to response headers for client debugging
        SpanContext spanContext = Span.current().getSpanContext();
        if (spanContext.isValid()) {
            response.getHeaders().putSingle("X-Trace-Id", spanContext.getTraceId());
        }
        // Clear MDC
        MDC.remove("traceId");
        MDC.remove("spanId");
    }
}
```

## 8. Jaeger Analysis Capabilities

Access Jaeger UI at http://localhost:16686 for:

| Feature | Description |
|---------|-------------|
| **Trace Search** | Find traces by service, operation, tags, and time range |
| **Trace Timeline** | Visualize span hierarchy and timing |
| **Service Map** | Automatic service dependency graph |
| **Compare Traces** | Side-by-side comparison of two traces |
| **Deep Dependency** | Analyze transitive service dependencies |
| **Latency Histogram** | P50, P95, P99 latency distributions |

## 9. Production Sampling Strategies

```properties
# Probabilistic Sampling (recommended for production)
quarkus.otel.traces.sampler=traceidratio
quarkus.otel.traces.sampler.arg=0.1  # Sample 10% of traces

# Parent-Based Sampling (respect upstream sampling decisions)
quarkus.otel.traces.sampler=parentbased_traceidratio
quarkus.otel.traces.sampler.arg=0.1

# Always Sample (development only)
quarkus.otel.traces.sampler=always_on

# Never Sample (disable tracing)
quarkus.otel.traces.sampler=always_off
```

## Validation Commands:

```bash
# Start infrastructure
docker compose up -d jaeger

# Start application
./mvnw quarkus:dev -pl container-app -am

# Generate test traffic
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","firstName":"John","lastName":"Doe"}'

# Verify trace headers in response
curl -v http://localhost:8080/api/v1/users/1 2>&1 | grep -i "x-trace-id"

# Access Jaeger UI
open http://localhost:16686

# Verify OTLP connectivity
curl -s http://localhost:16686/api/services | jq '.data'
```

# Additional Recommendations:

- **RECOMMENDED**: Implement adaptive sampling using OpenTelemetry's `ParentBasedSampler` with tail-based sampling in the collector — see OpenTelemetry Collector tail sampling processor (https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor).
- **RECOMMENDED**: Filter sensitive data from span attributes using an OpenTelemetry SpanProcessor — see OpenTelemetry SDK span processing (https://opentelemetry.io/docs/specs/otel/trace/sdk/#span-processor).
- **RECOMMENDED**: Use Jaeger's SPM (Service Performance Monitoring) for RED metrics derived from traces — see Jaeger SPM documentation (https://www.jaegertracing.io/docs/latest/spm/).
- **RECOMMENDED**: Configure Grafana alerting on trace-derived error rates and latency percentiles — see Grafana Alerting documentation (https://grafana.com/docs/grafana/latest/alerting/).
- **RECOMMENDED**: Configure Jaeger storage retention using `--es.max-span-age` for Elasticsearch or `--badger.span-store-ttl` for Badger — see Jaeger CLI flags (https://www.jaegertracing.io/docs/latest/cli/).
- **RECOMMENDED**: Secure Jaeger UI in production with reverse proxy authentication — see Jaeger security documentation (https://www.jaegertracing.io/docs/latest/deployment/#security).

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| OpenTelemetry Extension | `grep opentelemetry pom.xml` | Extension present |
| OTLP Exporter Config | `grep otel.exporter application.properties` | Exporter configured |
| Jaeger in Docker Compose | `grep jaeger docker-compose.yml` | Jaeger service defined |
| Trace Propagation | Response headers | X-Trace-Id header present |
| Custom Spans | Code inspection | @WithSpan annotations |
| Span Attributes | Code inspection | Span.current().setAttribute() calls |
| Log Correlation | Log output | traceId in JSON logs |
| Jaeger UI | `curl localhost:16686` | UI accessible |
| Service in Jaeger | Jaeger API | Service appears in dropdown |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Distributed Tracing Implementation..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for OpenTelemetry extension in pom.xml
if find . -name "pom.xml" -exec grep -q "quarkus-opentelemetry" {} + 2>/dev/null; then
  echo "✅ OpenTelemetry extension present in pom.xml"
else
  echo "❌ OpenTelemetry extension not found in any pom.xml"
  FAIL=1
fi

# Check for OTLP exporter configuration
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "quarkus.otel.exporter" "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ OTLP exporter configured"
else
  echo "❌ OTLP exporter not configured in application.properties"
  FAIL=1
fi

# Check for sampling configuration
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "quarkus.otel.traces.sampler" "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Sampling strategy configured"
else
  echo "❌ No sampling configuration in application.properties"
  FAIL=1
fi

# Check for Jaeger in Docker Compose
COMPOSE_FILE=$(ls docker-compose*.yml docker-compose.yaml compose.yml 2>/dev/null | head -1)
if [ -n "$COMPOSE_FILE" ] && grep -q "jaeger" "$COMPOSE_FILE"; then
  echo "✅ Jaeger service in Docker Compose"
else
  echo "❌ Jaeger service not in Docker Compose"
  FAIL=1
fi

# Check for @WithSpan annotations
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "@WithSpan" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ @WithSpan annotations found"
else
  echo "❌ No @WithSpan annotations found in source code"
  FAIL=1
fi

# Check for span attribute usage
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "Span.current().setAttribute\|@SpanAttribute" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Span attributes being set"
else
  echo "❌ No span attributes found in source code"
  FAIL=1
fi

# Check for trace context in log configuration
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "traceId" "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Trace ID correlation in log configuration"
else
  echo "❌ No traceId in log configuration (ADR-16 correlation)"
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
  echo "❌ Distributed Tracing validation failed"
  exit 1
fi
echo "✅ All Distributed Tracing criteria met"
```
