# Issue: Comprehensive Application Monitoring and Observability

Microservice architectures require comprehensive monitoring to ensure performance, security, and business intelligence. Without standardized dashboards and alerting, teams cannot track application health, identify bottlenecks, detect threats, or gather insights.

# Decision:

Implement comprehensive Grafana-based monitoring dashboards with Prometheus metrics providing complete visibility into application performance, security events, and business intelligence. All applications **MUST** implement standardized dashboard templates with automated alerting.

**Required Components**:
- **Application Performance**: HTTP requests, JVM metrics, worker pools, database connections  
- **Security Observability**: Authentication events, authorization failures, rate limiting, violations
- **Business Intelligence**: User operations, API usage patterns, validation metrics
- **Infrastructure Health**: System resources, container metrics, service availability

# Constraints:

- Monitoring overhead **MUST NOT** exceed 5% of application performance
- Metrics storage **MUST** support minimum 90-day retention for compliance
- Dashboard queries **MUST** complete within 5 seconds under normal load
- Critical alerts **MUST** be generated within 30 seconds of threshold breach
- Monitoring data **MUST NOT** expose sensitive information or credentials
- Dashboard access **MUST** be role-based and aligned with security policies

# Alternatives:

### 1. Commercial APM Solutions (New Relic, Datadog, AppDynamics)

Commercial APM platforms require proprietary agents and vendor-specific data formats. Datadog pricing starts at $15/host/month for infrastructure monitoring and $31/host/month for APM, with additional per-GB charges for log ingestion (https://www.datadoghq.com/pricing/). New Relic charges per-GB of ingested data beyond the free tier (https://newrelic.com/pricing). These platforms do not provide Quarkus-specific instrumentation; custom integration is required for Vert.x worker pool metrics and reactive SQL pool statistics that Micrometer exposes natively (https://quarkus.io/guides/telemetry-micrometer).

### 2. ELK Stack (Elasticsearch, Logstash, Kibana)

The ELK stack is designed primarily for log aggregation and search. Elasticsearch requires a minimum 3-node cluster for production resilience (https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability-cluster-small-clusters.html), consuming significantly more resources than a single Prometheus instance for time-series metrics. Prometheus's TSDB storage compresses metrics data at approximately 1.3 bytes per sample (https://prometheus.io/docs/prometheus/latest/storage/), while Elasticsearch indexes require substantially more storage per data point for equivalent metric queries.

# Rationale:

Quarkus provides the `quarkus-micrometer-registry-prometheus` extension which automatically exposes JVM metrics, HTTP server metrics, and reactive SQL pool statistics at the `/q/metrics` endpoint in Prometheus exposition format (https://quarkus.io/guides/telemetry-micrometer). This eliminates custom instrumentation for standard application metrics.

Prometheus scrapes the `/q/metrics` endpoint at configurable intervals, storing time-series data in its local TSDB with built-in compaction and retention management (https://prometheus.io/docs/prometheus/latest/storage/). PromQL enables alerting rules that evaluate against collected metrics, triggering notifications via Alertmanager within the configured evaluation interval (https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/).

Grafana connects to Prometheus as a data source and renders dashboard panels from PromQL queries. Grafana's provisioning API supports dashboard-as-code through JSON files and YAML datasource definitions, enabling version-controlled dashboard deployment (https://grafana.com/docs/grafana/latest/administration/provisioning/). The `quarkus-smallrye-health` extension provides `/q/health/live` and `/q/health/ready` endpoints for Kubernetes probe integration (https://quarkus.io/guides/smallrye-health).

# Implementation Guidelines:

## Core Monitoring Components:

### 1. Controller Statistics Dashboard:
- **HTTP Request Metrics**: Request rate via `http_server_bytes_written_count`, active requests, connection duration
- **REST API Performance**: Connection metrics, data transfer rates, concurrent request tracking
- **Load Distribution**: Request patterns, peak usage identification, throughput analysis

### 2. Quarkus Framework Statistics:
- **JVM Performance**: Memory usage (heap/non-heap), garbage collection metrics, thread utilization
- **Worker Pool Metrics**: Vert.x worker thread utilization, queue sizes, task execution timing
- **Database Connections**: Reactive SQL pool statistics, connection acquisition time, pool efficiency
- **Process Metrics**: Application uptime, startup time, CPU usage, file descriptor usage

### 3. Security Event Monitoring:
- **Authentication Events**: Login success/failure rates, session tracking (custom metrics required)
- **Authorization Failures**: Role-based access violations, endpoint access denials
- **Rate Limiting**: Request throttling events, IP-based blocking
- **Audit Trail**: User action tracking, data modification events

### 4. Business Intelligence Metrics:
- **Application Performance**: JVM metrics, GC performance, thread utilization
- **System Resource Usage**: CPU usage patterns, memory allocation trends
- **Database Performance**: Connection pool efficiency, query performance
- **Operational Metrics**: Application restarts, uptime trends, resource consumption
## Dashboard Organization:
```
/grafana/dashboards/
├── application-overview.json          # High-level application health
├── controller-performance.json        # REST API and controller metrics
├── quarkus-framework.json             # Framework-specific metrics
├── security-monitoring.json           # Security events and compliance
├── business-intelligence.json         # User behavior and API usage
└── infrastructure.json                # JVM, database, and system metrics
```

## CRITICAL: Grafana Configuration Requirements

### Datasource UID Configuration

Grafana datasources **MUST** have explicit UIDs defined in the provisioning configuration. Dashboard JSON files reference datasources by UID, and auto-generated UIDs **SHALL** cause dashboards to display "No data".

**Datasource Provisioning (`grafana/provisioning/datasources/datasources.yml`):**
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus           # REQUIRED: Explicit UID for dashboard references
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Jaeger
    type: jaeger
    uid: jaeger               # REQUIRED: Explicit UID for dashboard references
    access: proxy
    url: http://jaeger:16686
    editable: false
```

**Dashboard JSON Datasource Reference:**
```json
{
  "datasource": {
    "type": "prometheus",
    "uid": "prometheus"       // MUST match datasource provisioning UID
  }
}
```

### Dashboard Provisioning Structure

Dashboards **MUST** be provisioned correctly with separate paths for config and JSON files:

**Directory Structure:**
```
local-env/grafana/
├── provisioning/
│   ├── datasources/
│   │   └── datasources.yml      # Datasource definitions with explicit UIDs
│   └── dashboards/
│       └── dashboards.yml       # Dashboard provider configuration
└── dashboards/
    └── *.json                   # Actual dashboard JSON files
```

**Dashboard Provider Configuration (`grafana/provisioning/dashboards/dashboards.yml`):**
```yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards    # Path inside container
```

**Docker Compose Volume Mounts:**
```yaml
grafana:
  volumes:
    - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources:ro
    - ./grafana/provisioning/dashboards:/etc/grafana/provisioning/dashboards:ro
    - ./grafana/dashboards:/var/lib/grafana/dashboards:ro   # MUST be separate mount for JSON files
```

### Prometheus Scrape Target Configuration

When running the application in development mode (`make dev`) on the host machine, Prometheus **MUST** use `host.docker.internal` to reach the application. Using `localhost` or `app` **SHALL NOT** work from within a Docker container.

**Prometheus Configuration (`prometheus/prometheus.yml`):**
```yaml
scrape_configs:
  - job_name: 'copilot-quarkus'
    metrics_path: '/q/metrics'
    static_configs:
      # MUST use host.docker.internal for dev mode (app on host)
      - targets: ['host.docker.internal:8080']
      # SHOULD use app:8080 when running app in Docker container
      # - targets: ['app:8080']
```

### Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Dashboard shows "No data" | Datasource UID mismatch | **MUST** add explicit `uid` to datasource provisioning |
| Dashboard not appearing | Dashboard JSON not mounted | **MUST** mount dashboards to `/var/lib/grafana/dashboards` |
| Prometheus target down | Wrong target address | **MUST** use `host.docker.internal:8080` for dev mode |
| Changes not reflected | Grafana caching old config | **SHOULD** recreate container: `docker-compose up -d --force-recreate grafana` |

## Available Metrics:

### HTTP Server Metrics (Vert.x):
- `http_server_active_requests` - Current active HTTP requests
- `http_server_bytes_written_count` - Total HTTP requests (proxy metric)
- `http_server_connections_seconds_max` - Maximum connection duration

### JVM Metrics:
- `jvm_memory_used_bytes{area="heap|nonheap"}` - Memory usage by area
- `jvm_threads_live_threads` - Number of live threads
- `jvm_gc_pause_seconds_count{gc}` - GC event count by collector

### Worker Pool Metrics:
- `worker_pool_active{pool_name}` - Active worker threads
- `worker_pool_queue_size{pool_name}` - Worker pool queue size

### Database Pool Metrics:
- `sql_pool_active{pool_name}` - Active database connections
- `sql_pool_usage_seconds_max{pool_name}` - Maximum connection usage time

### Process Metrics:
- `process_uptime_seconds` - Application uptime
- `process_cpu_usage` - CPU usage by process

## Custom Metrics Implementation:

### Security Event Metrics:
```java
@ApplicationScoped
public class SecurityMetrics {
    @Inject MeterRegistry registry;
    
    public void recordAuthFailure(String reason) {
        Counter.builder("security_auth_failures_total")
            .tag("reason", reason)
            .register(registry).increment();
    }
    
    public void recordRateLimitViolation(String clientIp, String endpoint) {
        Counter.builder("security_rate_limit_violations_total")
            .tag("client_ip", clientIp).tag("endpoint", endpoint)
            .register(registry).increment();
    }
}
```

### Business Intelligence Metrics:
```java
@ApplicationScoped
public class BusinessMetrics {
    @Inject MeterRegistry registry;
    
    public void recordUserOperation(String operation, String entity) {
        Counter.builder("business_user_operations_total")
            .tag("operation", operation).tag("entity", entity)
            .register(registry).increment();
    }
    
    public void recordValidationFailure(String field, String error) {
        Counter.builder("business_validation_failures_total")
            .tag("field", field).tag("error", error)
            .register(registry).increment();
    }
}
```

## Alert Configuration:

### Critical Alerts:
- **Application Down**: `up{job="$job"} == 0`
- **High Request Queue**: `http_server_active_requests{job="$job"} > 50`
- **Memory Exhaustion**: `jvm_memory_used_bytes{job="$job",area="heap"} / jvm_memory_max_bytes{job="$job",area="heap"} > 0.95`
- **Database Pool Exhaustion**: `sql_pool_active{job="$job"} / (sql_pool_active{job="$job"} + sql_pool_idle{job="$job"}) > 0.95`

### Warning Alerts:
- **High Memory Usage**: `jvm_memory_used_bytes{job="$job",area="heap"} / jvm_memory_max_bytes{job="$job",area="heap"} > 0.80`
- **High CPU Usage**: `process_cpu_usage{job="$job"} > 0.80`
- **GC Pressure**: `rate(jvm_gc_pause_seconds_sum{job="$job"}[5m]) > 0.10`

### Security Alerts (Custom Metrics Required):
- **Authentication Failures**: `rate(security_auth_failures_total{job="$job"}[1m]) > 10`
- **Authorization Violations**: `rate(security_authorization_failures_total{job="$job"}[5m]) > 5`
- **Rate Limit Violations**: `rate(security_rate_limit_violations_total{job="$job"}[1m]) > 1`

## Validation Commands:

```bash
# Verify Grafana dashboard accessibility
curl -f http://localhost:3000/api/health

# Test Prometheus metrics endpoint
curl -f http://localhost:8080/q/metrics | grep -E "jvm_|http_|process_"

# Validate dashboard JSON structure
./mvnw exec:java -Dexec.mainClass="com.copilot.quarkus.monitoring.DashboardValidator"

# Test alerting configuration
curl -X POST http://localhost:3000/api/alerts/test -d '{"dashboard":"application-overview","rule":"high-memory"}'

# Verify metric collection rate
curl -s http://localhost:9090/api/v1/query?query=up | jq '.data.result[] | select(.metric.job=="quarkus-app")'
```

# Additional Recommendations:

- **RECOMMENDED**: Use Prometheus recording rules for complex calculations and query result caching — see Prometheus recording rules documentation (https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/).
- **RECOMMENDED**: Maintain dashboard version control with Grafana provisioning and Git — see Grafana provisioning guide (https://grafana.com/docs/grafana/latest/administration/provisioning/).
- **RECOMMENDED**: Implement custom HTTP request metrics using Micrometer `MeterFilter` for status code and route-level counters — see Micrometer concepts documentation (https://micrometer.io/docs/concepts#_meter_filters).
- **RECOMMENDED**: Connect Grafana with Jaeger data source for trace-to-metrics correlation — see Grafana Jaeger data source documentation (https://grafana.com/docs/grafana/latest/datasources/jaeger/).
- **RECOMMENDED**: Implement synthetic monitoring with Grafana k6 for critical API endpoint validation — see k6 documentation (https://grafana.com/docs/k6/latest/).

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Micrometer Extension | `grep micrometer pom.xml` | Micrometer present |
| Prometheus Endpoint | `curl localhost:8080/q/metrics` | Metrics exposed |
| Health Endpoint | `curl localhost:8080/q/health` | Health checks respond |
| Liveness Probe | `curl localhost:8080/q/health/live` | 200 OK |
| Readiness Probe | `curl localhost:8080/q/health/ready` | 200 OK |
| Custom Metrics | Code inspection | @Counted, @Timed annotations |
| Prometheus in Compose | `grep prometheus docker-compose.yml` | Prometheus defined |
| Grafana in Compose | `grep grafana docker-compose.yml` | Grafana defined |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Application Monitoring & Observability..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for Micrometer extension
if find . -name "pom.xml" -exec grep -q "micrometer\|smallrye-metrics" {} + 2>/dev/null; then
  echo "✅ Metrics extension present in pom.xml"
else
  echo "❌ Metrics extension (micrometer/smallrye-metrics) not found in pom.xml"
  FAIL=1
fi

# Check for SmallRye Health extension
if find . -name "pom.xml" -exec grep -q "smallrye-health" {} + 2>/dev/null; then
  echo "✅ Health check extension present in pom.xml"
else
  echo "❌ SmallRye Health extension not found in pom.xml"
  FAIL=1
fi

# Check for custom health checks
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "@Liveness\|@Readiness\|HealthCheck" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Custom health checks found"
else
  echo "❌ No custom health checks (@Liveness/@Readiness) found"
  FAIL=1
fi

# Check for custom metrics
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "@Counted\|@Timed\|@Gauge\|MeterRegistry" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Custom metrics annotations found"
else
  echo "❌ No custom metrics (@Counted/@Timed/MeterRegistry) found"
  FAIL=1
fi

# Check Docker Compose for monitoring stack
COMPOSE_FILE=$(ls docker-compose*.yml docker-compose.yaml compose.yml 2>/dev/null | head -1)
if [ -n "$COMPOSE_FILE" ]; then
  for service in prometheus grafana; do
    if grep -q "$service" "$COMPOSE_FILE"; then
      echo "✅ $service service in Docker Compose"
    else
      echo "❌ $service service not in Docker Compose"
      FAIL=1
    fi
  done
else
  echo "❌ No Docker Compose file found for monitoring stack"
  FAIL=1
fi

# Check for Grafana dashboard provisioning files
if find . -path "*/grafana/dashboards/*.json" -o -path "*/grafana/provisioning/datasources/*.yml" 2>/dev/null | grep -q .; then
  echo "✅ Grafana provisioning files found"
else
  echo "❌ No Grafana dashboard/datasource provisioning files found"
  FAIL=1
fi

# Check for Prometheus scrape configuration
if find . -name "prometheus.yml" -exec grep -q "metrics_path\|scrape_configs" {} + 2>/dev/null; then
  echo "✅ Prometheus scrape configuration found"
else
  echo "❌ No Prometheus scrape configuration (prometheus.yml) found"
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
  echo "❌ Application Monitoring & Observability validation failed"
  exit 1
fi
echo "✅ All Application Monitoring & Observability criteria met"
```
