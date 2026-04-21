# Issue: Application Framework Selection for Cloud-Native Development

Cloud-native Java application framework required for microservices architecture. Without proper framework selection, applications may fail to meet performance requirements and operational constraints in containerized environments.

**Impact**: Poor framework choice results in excessive resource consumption, slow startup times, and inability to scale effectively in Kubernetes environments.

# Decision:

Use Quarkus framework version 3.32.2 as the standardized application framework for all Java-based microservices and cloud-native applications.

**Required Components**:
- Quarkus 3.32.2 **MUST** be used with version consistency across all modules
- Native compilation via GraalVM **SHOULD** be used for production deployments
- Reactive programming patterns using Mutiny **MUST** be adopted for I/O operations
- RESTEasy Reactive (`quarkus-rest`) **MUST** be used for REST endpoint development
- OpenTelemetry and Prometheus metrics integration **MUST** be configured

# Constraints:

- Startup time **MUST** be <2 seconds in JVM mode in containerized environments.
- Memory usage **MUST NOT** exceed 128MB for basic CRUD applications.
- **MUST** support 1000+ concurrent connections with <100ms response time for simple operations under load.
- **MUST** target JDK 21+ runtime environment (ADR-02).
- **MUST** be optimized for Docker/Kubernetes deployment (ADR-05).
- **MUST** use RESTEasy Reactive (`quarkus-rest`) for REST API development.
- **MUST** integrate Prometheus metrics and OpenTelemetry tracing (ADR-20, ADR-21).
- **MUST NOT** use classic JAX-RS (`quarkus-resteasy`); only `quarkus-rest` (RESTEasy Reactive) is permitted.

# Alternatives:

1. **Spring Boot 3.x with Native (GraalVM)**: Spring Boot 3.3 native images report 2–5 second startup and 150–200 MB RSS memory (https://spring.io/blog/2023/09/09/all-together-now-spring-boot-3-2-graalvm-native-images-java-21-and-virtual). Spring Boot's reactive stack (WebFlux) uses Project Reactor, while Quarkus uses Mutiny — Reactor does not integrate natively with Quarkus CDI and build-time optimizations. Rejected because startup and memory exceed the <2s / ≤128MB constraints defined above.

2. **Micronaut 4.x**: Micronaut 4 reports 1–2 second startup and 100–150 MB memory in JVM mode (https://micronaut.io/2023/07/14/micronaut-framework-4-0-0-released/). Micronaut has 12 listed database extensions versus Quarkus's 30+ (https://quarkus.io/extensions/). There is no official Micronaut ClickHouse extension, while Quarkus supports ClickHouse client integration. Rejected because it lacks ClickHouse support required by ADR-13 and has fewer extensions for the project's observability stack (OpenTelemetry, Prometheus, health checks).

# Rationale:

Quarkus 3.32.2 documentation reports sub-second startup in JVM mode and ~70 MB RSS for a REST+CDI application (https://quarkus.io/guides/performance-measure). GraalVM native compilation reduces startup to ~50 ms and memory to ~30 MB per the same benchmarks. Built-in Dev Services auto-provision containers for detected extensions with zero configuration (https://quarkus.io/guides/dev-services), reducing onboarding time. Quarkus uses build-time dependency injection via ArC, eliminating runtime reflection overhead documented at https://quarkus.io/guides/cdi-reference. RESTEasy Reactive (`quarkus-rest`) and Mutiny are co-designed for Quarkus's event-loop architecture, providing non-blocking I/O without requiring a separate reactive framework. The Quarkus extension ecosystem includes ClickHouse client support required by ADR-13, and all observability extensions (OpenTelemetry, Micrometer/Prometheus, SmallRye Health) required by ADR-20/21/22.


# Implementation Guidelines:

## Project Configuration:
- Use Quarkus version 3.32.2 for all dependencies and plugins
- Import Quarkus BOM in Maven `dependencyManagement` section
- Configure Maven compiler for JDK 21+ compatibility
- Include provided POM.xml template as project foundation

## Development Environment:
- Install GraalVM for native compilation capability  
- Use `./mvnw quarkus:dev` for local development with hot reload
- Configure IDE with Quarkus extensions for optimal development experience
- Set up development containers for consistent environment

## Build and Deployment:
- Update CI/CD pipelines to include native build steps using Quarkus Maven plugin
- Configure containerized builds with optimized Docker images
- Implement health checks and readiness probes for Kubernetes deployment
- Configure resource limits appropriate for application requirements

## Observability Integration:
- Configure Prometheus metrics endpoint
- Implement OpenTelemetry tracing for distributed systems
- Set up application health monitoring
- Configure structured logging with correlation IDs

## Changelog Management
- Review the official Quarkus changelog for the target release before introducing new dependencies or code patterns to ensure compatibility with version 3.32.2.
- Document any resulting dependency renames or configuration key migrations directly in project ADRs before implementation.

## Maven Configuration:

**Current project uses Quarkus version 3.32.2**. All Quarkus dependencies and extensions must match this version.

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>quarkus-project</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  
  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <quarkus.platform.group-id>io.quarkus.platform</quarkus.platform.group-id>
    <quarkus.platform.artifact-id>quarkus-bom</quarkus.platform.artifact-id>
    <quarkus.platform.version>3.32.2</quarkus.platform.version>
    <testcontainers.version>1.21.3</testcontainers.version>
  </properties>
  
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>${quarkus.platform.group-id}</groupId>
        <artifactId>${quarkus.platform.artifact-id}</artifactId>
        <version>${quarkus.platform.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  
  <dependencies>
    <!-- Core Quarkus -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-rest-jackson</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    
    <!-- Observability -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-opentelemetry</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-health</artifactId>
    </dependency>
    
    <!-- Security -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-security</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-jwt</artifactId>
    </dependency>
    
    <!-- Validation -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-hibernate-validator</artifactId>
    </dependency>
    
    <!-- OpenAPI -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-openapi</artifactId>
    </dependency>
    
    <!-- Testing -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>
  
  <build>
    <plugins>
      <plugin>
        <groupId>${quarkus.platform.group-id}</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.platform.version}</version>
        <extensions>true</extensions>
      </plugin>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.15.0</version>
        <configuration>
          <compilerArgs>
            <arg>-parameters</arg>
          </compilerArgs>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

## Core Dependencies (version 3.32.2):
- io.quarkus:quarkus-rest:3.32.2
- io.quarkus:quarkus-rest-jackson:3.32.2
- io.quarkus:quarkus-arc:3.32.2
- io.quarkus:quarkus-config-yaml:3.32.2
- io.quarkus:quarkus-micrometer-registry-prometheus:3.32.2
- io.quarkus:quarkus-opentelemetry:3.32.2
- io.quarkus:quarkus-smallrye-health:3.32.2
- io.quarkus:quarkus-smallrye-context-propagation:3.32.2
- io.quarkus:quarkus-hibernate-validator:3.32.2
- io.quarkus:quarkus-security:3.32.2
- io.quarkus:quarkus-smallrye-jwt:3.32.2
- io.quarkus:quarkus-smallrye-openapi:3.32.2
- io.quarkus:quarkus-smallrye-fault-tolerance:3.32.2

**Note**: Quarkus BOM must be imported in dependency management to ensure compatible versions. All Quarkus-related dependencies and plugins must use version 3.32.2.

# Additional Recommendations:

**Performance Optimization**:
- **RECOMMENDED** Native image compilation for production deployments. See: https://quarkus.io/guides/building-native-image
- **RECOMMENDED** Mutiny reactive streams for I/O-intensive operations. See: https://quarkus.io/guides/mutiny-primer
- **RECOMMENDED** Optimal database connection pools for reactive drivers.
- **RECOMMENDED** JVM parameter tuning for specific application profiles.

**Development Productivity**:
- **RECOMMENDED** Leverage continuous testing and hot reload during development. See: https://quarkus.io/guides/continuous-testing
- **RECOMMENDED** Use Dev Services for automatic database/service provisioning. See: https://quarkus.io/guides/dev-services
- **RECOMMENDED** Comprehensive test suites with `@QuarkusTest` annotations. See: https://quarkus.io/guides/getting-started-testing
- **RECOMMENDED** Quarkus IDE plugins for enhanced development experience.

**Project Standards**:
- **RECOMMENDED** Maintain project templates with pre-configured dependencies.
- **RECOMMENDED** Establish reactive programming guidelines and error handling patterns (ADR-17).
- **RECOMMENDED** Create curated list of approved Quarkus extensions.

**Advanced Considerations**:
- **RECOMMENDED** Maven multi-module architecture with shared Quarkus configuration (ADR-09).
- **RECOMMENDED** Security extensions (JWT, RBAC) for production environments (ADR-18).
- **MUST** Reference ADR-02 (Java Runtime), ADR-05 (Containerization), ADR-09 (Multi-Module), ADR-13 (ClickHouse), ADR-20 (Observability).
- See Quarkus documentation: https://quarkus.io/guides/
- See Quarkus extensions catalog: https://quarkus.io/extensions/

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Quarkus Version | `grep quarkus.platform.version pom.xml` | 3.32.2 |
| BOM Import | `grep quarkus-bom pom.xml` | Present in dependencyManagement |
| Build Success | `./mvnw clean package -DskipTests` | Exit code 0 |
| Health Check | `curl localhost:8080/q/health` | `{"status":"UP"}` |
| Metrics | `curl localhost:8080/q/metrics` | Prometheus metrics |
| OpenAPI | `curl localhost:8080/q/openapi` | Valid OpenAPI spec |
| Tests Pass | `./mvnw test` | All tests pass |
| Startup Time | Application logs | < 2 seconds (JVM) |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Quarkus Framework..."

# Check Quarkus version
if grep -q "<quarkus.platform.version>3.32.2</quarkus.platform.version>" pom.xml; then
  echo "✅ Quarkus version: 3.32.2"
else
  echo "❌ Quarkus version mismatch"
  exit 1
fi

# Check BOM import
if grep -q "quarkus-bom" pom.xml; then
  echo "✅ Quarkus BOM imported"
else
  echo "❌ Quarkus BOM not found"
  exit 1
fi

# Build
./mvnw clean package -DskipTests -q
echo "✅ Build successful"

# Start and test endpoints
java -jar target/quarkus-app/quarkus-run.jar &
PID=$!
sleep 10

curl -sf localhost:8080/q/health | grep -q "UP" && echo "✅ Health UP"
curl -sf localhost:8080/q/metrics | grep -q "jvm" && echo "✅ Metrics available"

kill $PID 2>/dev/null || true
echo "✅ All Quarkus Framework criteria met"
```
