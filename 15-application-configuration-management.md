# Issue: Centralized Application Configuration Management

Quarkus microservices require comprehensive configuration management addressing security, observability, internationalization, database connectivity, and environment-specific deployment requirements. Without standardized templates, teams face inconsistent setups, security vulnerabilities, monitoring gaps, and deployment failures.

# Decision:

Implement standardized `application.properties` template consolidating all architectural decisions into comprehensive configuration file. All applications **MUST** use this template as foundation for environment-specific configuration with proper environment variable substitution and profile-based overrides.

**Required Coverage**:
- **Security Controls**: OWASP security headers, authentication, rate limiting, audit logging
- **Database Connectivity**: ClickHouse with proper connection management for analytical workloads
- **Observability Stack**: OpenTelemetry tracing, Micrometer metrics, structured logging
- **Internationalization**: Multi-locale configuration with proper fallback mechanisms
- **Environment Management**: Dev/test/prod profile configurations with secure defaults

## Complete application.properties Template:

```properties
# =============================================================================
# APPLICATION CONFIGURATION
# =============================================================================

# Application Information
quarkus.application.name=${project.artifactId:demo-app}
quarkus.application.version=${project.version:1.0.0-SNAPSHOT}

# Internationalization
quarkus.locales=en,es,fr,de
quarkus.default-locale=en

# Database Configuration
clickhouse.host=${CLICKHOUSE_HOST:localhost}
clickhouse.port=${CLICKHOUSE_PORT:8123}
clickhouse.database=${CLICKHOUSE_DB:demo}
clickhouse.username=${CLICKHOUSE_USER:demo}
clickhouse.password=${CLICKHOUSE_PASSWORD:demo}

# HTTP Configuration
quarkus.http.port=${HTTP_PORT:8080}
quarkus.http.host=0.0.0.0
quarkus.http.cors=true
quarkus.http.cors.origins=${CORS_ORIGINS:http://localhost:3000,http://localhost:8080}

# Security Headers
quarkus.http.header."X-Content-Type-Options".value=nosniff
quarkus.http.header."X-Frame-Options".value=DENY
quarkus.http.header."X-XSS-Protection".value=1; mode=block
quarkus.http.header."Referrer-Policy".value=strict-origin-when-cross-origin
quarkus.http.header."Content-Security-Policy".value=default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'

# OpenAPI/Swagger
quarkus.smallrye-openapi.path=/openapi
quarkus.swagger-ui.always-include=true
quarkus.swagger-ui.path=/swagger-ui

# Logging Configuration
quarkus.log.level=${LOG_LEVEL:INFO}
quarkus.log.category."com.copilot.quarkus".level=${APP_LOG_LEVEL:DEBUG}
%prod.quarkus.log.console.json=true
%prod.quarkus.log.console.json.date-format=uuuu-MM-dd'T'HH:mm:ss.SSSXXX
%dev.quarkus.log.console.json=false
quarkus.log.console.json.additional-field[traceId].value=%X{traceId}
quarkus.log.console.json.additional-field[correlationId].value=%X{correlationId}

# Observability
quarkus.micrometer.export.prometheus.enabled=true
quarkus.otel.enabled=${OTEL_ENABLED:true}
quarkus.otel.service.name=${quarkus.application.name}
quarkus.otel.exporter.otlp.traces.endpoint=${JAEGER_ENDPOINT:http://localhost:4317}

# Security Configuration
quarkus.security.users.embedded.enabled=${BASIC_AUTH_ENABLED:true}
quarkus.security.users.embedded.plain-text=true
quarkus.security.users.embedded.users.admin=${ADMIN_USER:admin}
quarkus.security.users.embedded.roles.admin=ADMIN,USER
mp.jwt.verify.publickey.location=${JWT_PUBLIC_KEY:META-INF/resources/publicKey.pem}
mp.jwt.verify.issuer=${JWT_ISSUER:https://demo-app}

# Rate Limiting
app.security.rate-limit.enabled=${RATE_LIMIT_ENABLED:true}
app.security.rate-limit.max-requests=${RATE_LIMIT_MAX:100}
app.security.rate-limit.window-duration=${RATE_LIMIT_WINDOW:PT1M}

# Development Profile
%dev.quarkus.http.cors.origins=http://localhost:3000,http://localhost:8080,http://localhost:5173
%dev.quarkus.log.category."com.copilot.quarkus".level=DEBUG
%dev.quarkus.hibernate-orm.database.generation=drop-and-create

# Test Profile (uses Testcontainers — see ADR-28)
%test.quarkus.hibernate-orm.database.generation=none

# Production Profile
%prod.quarkus.log.level=INFO
%prod.quarkus.hibernate-orm.database.generation=none
%prod.quarkus.security.users.embedded.enabled=false
%prod.quarkus.http.insecure-requests=disabled
```

## Environment Variable Guidelines:

### Required for Production:
```bash
# Database
CLICKHOUSE_HOST=prod-clickhouse
CLICKHOUSE_PORT=8123
CLICKHOUSE_DB=production_db
CLICKHOUSE_USER=production_user
CLICKHOUSE_PASSWORD=secure_password

# Security
JWT_ISSUER=https://your-domain.com
ADMIN_USER=admin

# Observability  
JAEGER_ENDPOINT=http://jaeger:4317
SERVICE_NAMESPACE=production
LOG_LEVEL=INFO
```

### Optional Variables:
```bash
# Features
RATE_LIMIT_ENABLED=true
OTEL_ENABLED=true
RATE_LIMIT_MAX=1000
OTEL_SAMPLE_RATIO=0.1
```

# Constraints:

- **MUST NOT** deploy applications without mandatory security configurations enabled
- **MUST NOT** include production secrets in configuration files committed to version control
- **MUST NOT** allow production configurations to impact development environments  
- **MUST NOT** disable security controls by default in any profile
- Configuration **MUST** support automated CI/CD deployments without manual intervention
- Connection pooling and timeout configurations **MUST** be optimized for cloud deployments

# Alternatives:

**YAML Configuration**: Quarkus supports `application.yml` via the `quarkus-config-yaml` extension. The properties format is the default in all Quarkus guides and the `quarkus create` CLI scaffolding (https://quarkus.io/guides/config-reference). The `${VAR:default}` substitution syntax works identically in both formats, but the properties format requires no additional extension dependency. Rejected because adding an extra extension provides no functional benefit for this project's key-value configuration.

**External Configuration Services (Consul/Spring Cloud Config)**: HashiCorp Consul and Spring Cloud Config require a separate running server process and network round-trips at startup (https://developer.hashicorp.com/consul/docs/dynamic-app-config/kv). Quarkus natively reads environment variables and Kubernetes ConfigMaps/Secrets via MicroProfile Config without additional infrastructure (https://quarkus.io/guides/config-reference#configuring-quarkus). Rejected because Kubernetes-native environment variable injection satisfies the project's configuration requirements without an external service dependency.

# Rationale:

Quarkus MicroProfile Config resolves properties in a defined ordinal hierarchy: `application.properties` → environment variables → system properties (https://quarkus.io/guides/config-reference#configuration-sources). The `${VAR:default}` syntax allows every secret and environment-specific value to be injected at deploy time without modifying the artifact. Profile prefixes (`%dev.`, `%test.`, `%prod.`) are evaluated at startup based on `quarkus.profile`, eliminating the need for per-environment property files (https://quarkus.io/guides/config-reference#profiles). Security headers are set as static response headers via `quarkus.http.header.*`, which Quarkus applies to every HTTP response without application code (https://quarkus.io/guides/http-reference#additional-http-headers).

# Implementation Guidelines:

## Configuration Setup:

1. **Base Template Implementation**:
   - Copy complete template as foundation for new Quarkus applications
   - Place `application.properties` in `src/main/resources/` directory
   - Validate all required sections present before deployment

2. **Environment Variable Configuration**:
   - Use `${VARIABLE:default_value}` pattern for all configurable values
   - Provide sensible defaults for development environment
   - Document all environment variables in project README

3. **Profile-Specific Configuration**:
   - Use `%profile.property=value` format for environment-specific overrides
   - Development profile enables debugging and verbose logging
   - Production profile disables development features and optimizes performance
   - Test profile uses in-memory databases and isolated configurations

## Environment Variables:

### Required for Production:
```bash
# Database Configuration
export CLICKHOUSE_HOST=prod-clickhouse
export CLICKHOUSE_PORT=8123
export CLICKHOUSE_DB=production_db
export CLICKHOUSE_USER=production_user
export CLICKHOUSE_PASSWORD=secure_password_here

# Security Configuration  
export JWT_ISSUER=https://your-production-domain.com
export ADMIN_USER=production_admin

# Observability
export JAEGER_ENDPOINT=http://jaeger:4317
export SERVICE_NAMESPACE=production
export LOG_LEVEL=INFO
export HTTP_PORT=8080
```

### Optional Variables:
```bash
# Feature Flags
export RATE_LIMIT_ENABLED=true
export OTEL_ENABLED=true
export RATE_LIMIT_MAX=1000
export OTEL_SAMPLE_RATIO=0.1
export CORS_ORIGINS=https://your-frontend.com
```

## Validation Commands:

```bash
# Verify application starts successfully
./mvnw quarkus:dev

# Test security headers are configured
curl -I http://localhost:8080/q/health | grep -E "X-Content-Type-Options|X-Frame-Options"

# Verify OpenAPI documentation accessible
curl http://localhost:8080/q/openapi | jq .info.title

# Test observability endpoints
curl http://localhost:8080/q/metrics | grep application_
curl http://localhost:8080/q/health | jq .status
```

# Additional Recommendations:

- It is RECOMMENDED to validate required configuration at startup using `@ConfigProperty` with `Optional` or build-time validation (https://quarkus.io/guides/config-reference#inject)
- It is RECOMMENDED to use HashiCorp Vault or AWS Secrets Manager for production secrets via the `quarkus-vault` extension (https://quarkiverse.github.io/quarkiverse-docs/quarkus-vault/dev/index.html)
- It is RECOMMENDED to create `.env` files for Docker Compose local development (see ADR-05, ADR-07)
- Configuration template **SHOULD** be updated when new ADR decisions require additional properties (see ADR-00)
- It is RECOMMENDED to integrate configuration validation into CI/CD pipelines using `./mvnw quarkus:build` which fails on missing required config (https://quarkus.io/guides/config-reference#configuration-aware-build)

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| application.properties Exists | `ls src/main/resources/application.properties` | File exists |
| Profile Configs | `grep '%dev\|%test\|%prod' application.properties` | Profile-prefixed properties present |
| Environment Variables | `grep '\${' application.properties` | Env vars used for secrets |
| No Hardcoded Secrets | `grep -i password application*.properties` | No plaintext passwords |
| Config Injection Works | Unit test | @ConfigProperty injects values |
| Profile Activation | `./mvnw quarkus:dev -Dquarkus.profile=dev` | Dev profile loads |
| Build Success | `./mvnw clean compile` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Application Configuration Management..."

FAIL=0

# Check for main application.properties
if find . -path "*/src/main/resources/application.properties" | grep -q .; then
  echo "✅ application.properties exists"
else
  echo "❌ application.properties not found"
  FAIL=1
fi

# Check for profile-prefixed properties (dev, test, prod)
PROFILES=("dev" "test" "prod")
for profile in "${PROFILES[@]}"; do
  if grep -rq "%${profile}\." */src/main/resources/application*.properties 2>/dev/null; then
    echo "✅ Profile '$profile' properties present"
  else
    echo "❌ Profile '$profile' properties not found"
    FAIL=1
  fi
done

# Check for environment variable usage (good practice for secrets)
if grep -rq '\${' */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ Environment variable substitution detected"
else
  echo "❌ No environment variable substitution found"
  FAIL=1
fi

# Check for hardcoded passwords (bad practice)
HARDCODED=$(grep -rn "password\s*=\s*[^$\{]" */src/main/resources/application*.properties 2>/dev/null | grep -v '#' | head -3)
if [ -z "$HARDCODED" ]; then
  echo "✅ No hardcoded passwords detected"
else
  echo "❌ Possible hardcoded passwords: $HARDCODED"
  FAIL=1
fi

# Check Hibernate DDL is none in prod profile
if grep -rq '%prod.*hibernate-orm.database.generation=none' */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ Production Hibernate DDL set to none"
else
  echo "❌ Production Hibernate DDL is not set to none (see ADR-14)"
  FAIL=1
fi

# Compile to verify config is valid
./mvnw clean compile -q && echo "✅ Configuration compiles successfully" || { echo "❌ Build failed"; FAIL=1; }

if [ $FAIL -ne 0 ]; then
  echo "❌ Application Configuration Management validation FAILED"
  exit 1
fi
echo "✅ All Application Configuration Management criteria met"
```
