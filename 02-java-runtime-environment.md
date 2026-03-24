# Issue: Java Runtime Environment Standardization

Lack of standardized Java runtime environment creates inconsistencies across development, testing, and production environments.

# Decision:

Use Java Development Kit (JDK) version 21 or higher as the standard runtime environment.

All development, testing, and production environments utilize only stable, production-ready features while excluding preview or experimental functionality.

# Constraints:
- **MUST** Use SDKMAN for cross-platform JDK version management
- **MUST** Maintain compatibility with Quarkus framework requirements (minimum JDK 17 for Quarkus 3.x)
- **MUST NOT** use preview or experimental features in production code
- **MUST NOT** use JDK versions below 21 for new development

# Alternatives:

**GraalVM CE/EE as Primary Runtime**: Rejected. GraalVM native image requires closed-world analysis, meaning all classes must be known at build time (documented in [GraalVM Native Image Basics](https://www.graalvm.org/latest/reference-manual/native-image/basics/)). Reflection-heavy libraries require manual metadata configuration via `reflect-config.json` (documented in [GraalVM Reflection Support](https://www.graalvm.org/latest/reference-manual/native-image/metadata/)). Native image build times typically add 2-5 minutes per build compared to standard JVM compilation (documented in [Quarkus Native Build Guide](https://quarkus.io/guides/building-native-image)). Standard JDK 21 with Quarkus achieves sub-second startup in JVM mode (documented in [Quarkus Performance Benchmarks](https://quarkus.io/guides/performance-measure)). GraalVM native image support can be explored as an optional optimization layer without replacing the primary runtime.

# Rationale:

- JDK 21 is designated as Long-Term Support (LTS) with Oracle Premier Support until September 2028 and Extended Support until September 2031 ([Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)).
- Virtual threads ([JEP 444](https://openjdk.org/jeps/444)) eliminate the need for thread-pool sizing by allowing millions of concurrent threads with minimal OS thread usage, reducing blocking I/O latency in high-concurrency scenarios.
- Pattern matching for switch ([JEP 441](https://openjdk.org/jeps/441)) and record patterns ([JEP 440](https://openjdk.org/jeps/440)) reduce type-checking boilerplate by eliminating explicit casting and nested `instanceof` checks.
- Generational ZGC ([JEP 439](https://openjdk.org/jeps/439)) targets sub-millisecond max pause times regardless of heap size, documented in the JEP specification, which directly benefits container workloads with strict latency requirements.
- Quarkus 3.x documents full JDK 21 support including virtual threads integration ([Quarkus Virtual Threads Guide](https://quarkus.io/guides/virtual-threads)), with measured startup times under 1 second in JVM mode and memory footprint under 100MB RSS ([Quarkus Performance Guide](https://quarkus.io/guides/performance-measure)).

# Implementation Guidelines:

## Mandatory Setup Steps:

1. **JDK Installation**:
   - Use SDKMAN for cross-platform JDK version management (preferred approach):
     ```bash
     # Install SDKMAN
     curl -s "https://get.sdkman.io" | bash
     source "$HOME/.sdkman/bin/sdkman-init.sh"
     
     # Install and set JDK 21
     sdk install java 21.0.4-tem
     sdk use java 21.0.4-tem
     sdk default java 21.0.4-tem
     ```
   - Alternative: Use platform package managers (brew, apt, yum) or direct downloads
   - Install JDK 21 or higher using chosen method

2. **Environment Configuration**:
   - Set JAVA_HOME environment variable to JDK 21 installation path
   - Verify installation with `java -version` showing version 21+
   - Verify compilation with `javac -version` showing version 21+

3. **IDE Configuration**:
   - Configure IDE (IntelliJ IDEA, VS Code, Eclipse) to use JDK 21 for project SDK
   - Set project language level to 21 to enable all language features
   - Configure compiler compliance level to 21

4. **Build System Integration**:
   - Update Maven `pom.xml` or Gradle `build.gradle` to specify JDK 21:
     ```xml
     <properties>
         <maven.compiler.source>21</maven.compiler.source>
         <maven.compiler.target>21</maven.compiler.target>
     </properties>
     ```

5. **Container Configuration**:
   - Update Dockerfiles to use JDK 21 base images: `FROM openjdk:21-jdk-slim`
   - Use multi-stage builds for optimized production images
   - Validate container Java version in CI/CD pipelines

6. **Documentation and Onboarding**:
   - Document JDK installation process in project README
   - Create setup scripts for automated environment configuration
   - Include JDK version verification in onboarding checklist

## Validation Steps:
- Run `java -version` and verify output shows "openjdk version \"21.x.x\""
- Execute `mvn clean compile` (or `gradle build`) without errors
- Start Quarkus application in dev mode: `mvn quarkus:dev`
- Verify container builds successfully with new JDK version

# Additional Recommendations:

## References:
- **JDK 21 Release Notes**: https://openjdk.org/projects/jdk/21/
- **SDKMAN Documentation**: https://sdkman.io/usage
- **Quarkus JDK Compatibility**: https://quarkus.io/guides/building-native-image
- **Eclipse Temurin Downloads**: https://adoptium.net/temurin/releases/
- **JVM Container Best Practices**: https://developers.redhat.com/articles/2024/07/09/best-practices-java-containerized-environments

## Performance Optimization:
- **JVM Tuning**: Configure JVM parameters for containerized environments:
  ```bash
  -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0
  ```

## Development Productivity:
- **SDKMAN for Version Management**: Strongly recommended for team consistency:
  ```bash
  # List available JDK versions
  sdk list java
  
  # Switch between JDK versions per project
  sdk use java 21.0.4-tem
  
  # Set global default
  sdk default java 21.0.4-tem
  ```
- **Quarkus Dev Services**: Leverage automatic dev service provisioning for databases and message brokers during development
- **Hot Reload**: Utilize Quarkus continuous testing and hot reload capabilities enabled by JDK 21 performance improvements
- **Alternative Version Managers**: May use jenv on macOS/Linux or Chocolatey on Windows if SDKMAN is not available


# Success Criteria:

> **Completion Gate**: The following criteria **MUST** all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| JDK Version | `java -version 2>&1 \| head -1` | Output contains `21.0.4` or higher |
| SDKMAN Configuration | `cat .sdkmanrc` | File exists with `java=21.0.4-tem` |
| Maven Compiler Source | `grep maven.compiler.source pom.xml` | Value is `21` |
| Maven Compiler Target | `grep maven.compiler.target pom.xml` | Value is `21` |
| Compilation Success | `./mvnw clean compile` | Exit code 0, no errors |
| Quarkus Dev Mode | `./mvnw quarkus:dev` | Application starts without JDK errors |
| Virtual Threads Available | Code using `Thread.ofVirtual()` | Compiles and executes |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating JDK 21 Environment..."

# Check Java version
JAVA_VERSION=$(java -version 2>&1 | head -n 1 | cut -d'"' -f2)
if [[ "$JAVA_VERSION" != 21.0.* ]]; then
  echo "❌ Java version $JAVA_VERSION detected. Required: 21.0.x"
  exit 1
fi
echo "✅ Java version: $JAVA_VERSION"

# Verify Maven compiler settings
if grep -q "<maven.compiler.source>21</maven.compiler.source>" pom.xml; then
  echo "✅ Maven compiler configured for JDK 21"
else
  echo "❌ Maven compiler source not set to 21"
  exit 1
fi

# Test compilation
./mvnw clean compile -q
echo "✅ Project compiles successfully with JDK 21"

echo "✅ All JDK 21 criteria met"
```
