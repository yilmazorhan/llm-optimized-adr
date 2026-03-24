# Issue: Observability Strategy for Cloud-Native Microservices

Distributed microservices without unified observability cause fragmented monitoring across services, inability to correlate events across service boundaries, increased Mean Time To Detection (MTTD) and Mean Time To Resolution (MTTR), and no end-to-end visibility for request flows.

# Decision:

Implement a comprehensive Three Pillars of Observability strategy integrating **Metrics**, **Logs**, and **Traces** into a unified observability platform. All microservices **MUST** implement consistent observability patterns enabling end-to-end visibility across the distributed system.

**Required Components**:
- **Metrics**: Prometheus-based metrics collection with Micrometer integration (see ADR-22)
- **Traces**: OpenTelemetry-based distributed tracing with Jaeger backend (see ADR-21)
- **Logs**: Structured JSON logging with correlation IDs (see ADR-16)
- **Visualization**: Grafana dashboards for unified observability (see ADR-22)
- **Correlation**: Trace ID propagation across all three pillars

# Constraints:

- Observability infrastructure **MUST NOT** add more than 5% overhead to application performance
- All observability components **MUST** work with GraalVM native compilation
- Observability data **MUST NOT** contain sensitive information (PII, credentials, tokens)
- **MUST** support automatic context propagation across reactive Mutiny streams
- **MUST** integrate seamlessly with Kubernetes and container orchestration platforms
- Production environments **MUST** implement sampling strategies to control data volume and costs

# Alternatives:

### 1. Commercial APM Solutions (Datadog, New Relic, Dynatrace)

Commercial APM platforms provide integrated observability but require proprietary agents and data formats. Datadog's pricing is per-host per-month with additional costs per custom metric and per indexed log GB (https://www.datadoghq.com/pricing/). These platforms do not support OpenTelemetry-native export without vendor-specific adapters, creating lock-in that contradicts the CNCF vendor-neutral observability recommendation (https://www.cncf.io/blog/2024/01/04/opentelemetry-in-2024/). Quarkus provides zero built-in integration for Datadog or New Relic agents; custom extensions would be required, adding maintenance overhead outside the supported extension ecosystem (https://quarkus.io/extensions/).

### 2. Separate Unintegrated Open Source Tools (e.g., StatsD + Syslog + Zipkin)

Using disconnected tools for each pillar (e.g., StatsD for metrics, syslog for logs, Zipkin for traces) eliminates cross-pillar correlation. Without a shared trace context propagator, correlating a metric anomaly to the originating trace requires manual timestamp matching. OpenTelemetry's unified SDK provides automatic context propagation across all three signals through a single dependency (https://opentelemetry.io/docs/concepts/signals/), which disconnected tools cannot replicate without custom bridging code.

# Rationale:

OpenTelemetry is the CNCF-graduated project for observability instrumentation, providing a single SDK that emits metrics, logs, and traces with shared context propagation (https://opentelemetry.io/docs/what-is-opentelemetry/). Quarkus integrates OpenTelemetry through the `quarkus-opentelemetry` extension, which automatically instruments RESTEasy Reactive endpoints, gRPC calls, and Mutiny reactive pipelines without application code changes (https://quarkus.io/guides/opentelemetry).

Prometheus metrics collection uses the `quarkus-micrometer-registry-prometheus` extension, which exposes a `/q/metrics` endpoint compatible with Prometheus scrape configuration (https://quarkus.io/guides/telemetry-micrometer). Jaeger receives traces via the OTLP protocol, supporting both gRPC (port 4317) and HTTP (port 4318) exporters (https://www.jaegertracing.io/docs/latest/apis/#opentelemetry-protocol-stable).

Grafana provides unified visualization across all three pillars by connecting to Prometheus, Jaeger, and Loki as data sources, enabling cross-signal correlation through shared trace IDs (https://grafana.com/docs/grafana/latest/datasources/).

# Implementation Guidelines:

## Observability Architecture Overview:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Quarkus Microservice                        │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Metrics   │  │    Logs     │  │        Traces           │  │
│  │ (Micrometer)│  │ (SLF4J/JSON)│  │    (OpenTelemetry)      │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
│         │                │                     │                │
│         │    ┌───────────┴───────────┐         │                │
│         │    │   Correlation ID      │         │                │
│         │    │   (traceId/spanId)    │         │                │
│         │    └───────────────────────┘         │                │
└─────────┼────────────────┼─────────────────────┼────────────────┘
          │                │                     │
          ▼                ▼                     ▼
    ┌───────────┐    ┌───────────┐        ┌───────────┐
    │Prometheus │    │   Loki/   │        │  Jaeger   │
    │           │    │   ELK     │        │           │
    └─────┬─────┘    └─────┬─────┘        └─────┬─────┘
          │                │                     │
          └────────────────┼─────────────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Grafana   │
                    │ (Unified UI)│
                    └─────────────┘
```

## Correlation Strategy:

All three pillars **MUST** share the same correlation identifiers:

```java
// Example: Unified correlation in a service method
@WithSpan("UserService.createUser")
public Uni<User> createUser(CreateUserCommand command) {
    // Trace context is automatically propagated
    String traceId = Span.current().getSpanContext().getTraceId();
    String spanId = Span.current().getSpanContext().getSpanId();
    
    // Add to MDC for structured logging
    MDC.put("traceId", traceId);
    MDC.put("spanId", spanId);
    
    // Add custom span attributes
    Span.current().setAttribute("user.email", command.getEmail());
    
    // Log with correlation (automatically includes traceId from MDC)
    logger.info("Creating user with email: {}", command.getEmail());
    
    // Business logic...
    return userRepository.create(user)
        .invoke(created -> {
            // Custom metric
            userCreatedCounter.increment();
            logger.info("User created with ID: {}", created.getId());
        });
}
```

## Configuration Integration:

```properties
# =============================================================================
# UNIFIED OBSERVABILITY CONFIGURATION
# =============================================================================

# Application Identity (shared across all pillars)
quarkus.application.name=copilot-quarkus-demo
quarkus.application.version=1.0.0

# --- METRICS (ADR-22) ---
quarkus.micrometer.export.prometheus.enabled=true
quarkus.micrometer.binder.jvm=true
quarkus.micrometer.binder.http-server.enabled=true

# --- TRACING (ADR-21) ---
quarkus.otel.enabled=true
quarkus.otel.service.name=${quarkus.application.name}
quarkus.otel.exporter.otlp.traces.endpoint=http://localhost:4317
quarkus.otel.traces.sampler=traceidratio
quarkus.otel.traces.sampler.arg=1.0

# --- LOGGING (ADR-16) ---
%prod.quarkus.log.console.json=true
%prod.quarkus.log.console.json.date-format=uuuu-MM-dd'T'HH:mm:ss.SSSXXX
quarkus.log.console.json.additional-field[traceId].value=%X{traceId}
quarkus.log.console.json.additional-field[spanId].value=%X{spanId}

# --- HEALTH CHECKS ---
quarkus.smallrye-health.root-path=/q/health
quarkus.health.extensions.enabled=true
```

## Docker Compose - Complete Observability Stack:

```yaml
services:
  # Application
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      QUARKUS_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT: http://jaeger:4317
    depends_on:
      - jaeger
      - prometheus

  # Distributed Tracing (ADR-21)
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: true

  # Metrics Collection (ADR-22)
  prometheus:
    image: prom/prometheus:v2.54.1
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  # Visualization (ADR-22)
  grafana:
    image: grafana/grafana:11.1.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
```

## Observability Endpoints:

| Endpoint | Purpose | Pillar |
|----------|---------|--------|
| `/q/metrics` | Prometheus metrics | Metrics |
| `/q/health` | Health checks | Metrics |
| `/q/health/live` | Liveness probe | Metrics |
| `/q/health/ready` | Readiness probe | Metrics |
| `stdout/stderr` | JSON structured logs | Logs |
| Jaeger UI `:16686` | Trace exploration | Traces |
| Grafana `:3000` | Unified dashboards | All |

## Related ADRs:

- **ADR-16**: Structured Logging Strategy - Defines JSON logging format and correlation ID propagation
- **ADR-21**: Distributed Tracing Strategy - OpenTelemetry and Jaeger implementation details
- **ADR-22**: Application Monitoring & Observability - Prometheus metrics and Grafana dashboards

# Additional Recommendations:

- **RECOMMENDED**: Implement Service Level Objectives (SLOs) using Prometheus recording rules and Grafana SLO dashboards — see Grafana SLO documentation (https://grafana.com/docs/grafana-cloud/alerting-and-irm/slo/).
- **RECOMMENDED**: Create runbooks linking alerts to observability dashboards for incident response — see Grafana OnCall runbook integration (https://grafana.com/docs/oncall/latest/configure/integrations/).
- **RECOMMENDED**: Configure observability data retention policies in Prometheus (`--storage.tsdb.retention.time`) and Jaeger — see Prometheus storage documentation (https://prometheus.io/docs/prometheus/latest/storage/).
- **RECOMMENDED**: Use Grafana alerting with anomaly detection for proactive issue identification — see Grafana Alerting documentation (https://grafana.com/docs/grafana/latest/alerting/).
- **RECOMMENDED**: Review Quarkus Dev Services for automatic observability stack provisioning during development — see Quarkus Dev Services guide (https://quarkus.io/guides/dev-services).

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Metrics Endpoint | `curl localhost:8080/q/metrics` | Prometheus metrics returned |
| Health Endpoint | `curl localhost:8080/q/health` | Health status returned |
| Trace Export | Jaeger UI | Traces visible in Jaeger |
| Log Correlation | Log inspection | traceId in JSON logs |
| Grafana Access | `curl localhost:3000` | Grafana UI accessible |
| Prometheus Scrape | Prometheus targets | App target UP |
| OpenTelemetry Config | `grep otel application.properties` | OTel configured |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Observability Strategy..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for OpenTelemetry configuration
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "quarkus.otel" "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ OpenTelemetry configured"
else
  echo "❌ OpenTelemetry not configured in application.properties"
  FAIL=1
fi

# Check for Micrometer/Prometheus metrics dependency
if find . -name "pom.xml" -exec grep -q "micrometer\|quarkus-smallrye-metrics" {} + 2>/dev/null; then
  echo "✅ Metrics extension present in pom.xml"
else
  echo "❌ Metrics extension (micrometer/smallrye-metrics) not found in pom.xml"
  FAIL=1
fi

# Check for JSON logging configuration
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "quarkus.log.console.json" "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ JSON logging configured"
else
  echo "❌ JSON logging not configured in application.properties"
  FAIL=1
fi

# Check for trace ID correlation in logs
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "traceId\|correlationId" "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Log correlation (traceId) configured"
else
  echo "❌ Log correlation (traceId/correlationId) not configured"
  FAIL=1
fi

# Check for health check extension
if find . -name "pom.xml" -exec grep -q "smallrye-health" {} + 2>/dev/null; then
  echo "✅ SmallRye Health extension present"
else
  echo "❌ SmallRye Health extension not found in pom.xml"
  FAIL=1
fi

# Check Docker Compose for observability stack
COMPOSE_FILE=$(ls docker-compose*.yml docker-compose.yaml compose.yml 2>/dev/null | head -1)
if [ -n "$COMPOSE_FILE" ]; then
  for service in jaeger prometheus grafana; do
    if grep -q "$service" "$COMPOSE_FILE"; then
      echo "✅ $service service in Docker Compose"
    else
      echo "❌ $service service not in Docker Compose"
      FAIL=1
    fi
  done
else
  echo "❌ No Docker Compose file found for observability stack"
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
  echo "❌ Observability Strategy validation failed"
  exit 1
fi
echo "✅ All Observability Strategy criteria met"
```
