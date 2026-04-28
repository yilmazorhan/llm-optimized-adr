# Issue: Build Automation Tool Configuration

Inconsistent build tools and versions across team members create "works on my machine" issues, build failures, and deployment inconsistencies.

This affects dependency resolution, CI/CD pipeline execution, and new developer onboarding across all build processes.

# Decision:

Use Apache Maven as the standard build automation tool for all project components.

Maven Wrapper (mvnw) configuration **MUST** be present and functional before any code generation, development, or project setup.

**Requirements:**
- Apache Maven 3.9.x or higher **MUST** be used
- Maven Wrapper (mvnw) **MUST** be configured and committed to repository
- All environments **MUST** use identical Maven versions via wrapper
- All build commands **MUST** use Maven Wrapper (`./mvnw`)
- Maven Wrapper **MUST** be validated before any development activities

# Ownership

| Role | Person | Competencies |
|------|--------|-------------|
| **Responsible** | DevOps / Platform Engineer | Maven Wrapper configuration and distribution, CI/CD pipeline build integration, dependency management strategies, multi-module Maven project structure, build reproducibility enforcement |
| **Approver** | Tech Lead / Software Architect | Build tool selection trade-offs (Maven vs. Gradle), Quarkus framework tooling ecosystem assessment, dependency governance policies, cross-team build standardization decisions |

# Constraints:

## Technical:
- Maven Wrapper (mvnw) **MUST** be present and functional before any development activities
- **MUST NOT** use system-wide Maven installations for project builds
- **MUST NOT** allow different Maven versions across team members
- **MUST NOT** proceed with code generation without validated Maven Wrapper
- **MUST NOT** exclude `.mvn/wrapper/maven-wrapper.jar` from version control — the JAR is a required component for the wrapper to function
- All dependency versions **MUST** be defined in the `<properties>` section of the parent `pom.xml` and referenced using property placeholders (e.g., `<version>${clickhouse.version}</version>`). **MUST NOT** hardcode version numbers directly in `<dependency>` or `<plugin>` declarations outside `<dependencyManagement>` or `<pluginManagement>`. This ensures a single source of truth for all versions, prevents version drift across modules, and simplifies dependency upgrades.

# Alternatives:

**Gradle**: Rejected. **MUST** avoid.

Quarkus official documentation and code generator (`code.quarkus.io`) use Maven as the default build tool (https://quarkus.io/guides/maven-tooling). The Quarkus Maven plugin is maintained by the core team and receives updates before the Gradle plugin. The Quarkus Gradle plugin documentation (https://quarkus.io/guides/gradle-tooling) notes that multi-module dev mode requires additional configuration and has known limitations compared to the Maven plugin.

**System-wide Maven Installation**: Rejected. **MUST** avoid.

System-wide installation creates version inconsistencies across team members. The Maven Wrapper documentation (https://maven.apache.org/wrapper/) specifies that the wrapper pins a specific Maven distribution URL per project, ensuring all developers and CI systems use the exact same version without relying on locally installed Maven.

**MAY** be used as fallback for Maven Wrapper creation only.

# Rationale:

Quarkus official guides, CLI, and `code.quarkus.io` default to Maven project structure (https://quarkus.io/guides/maven-tooling). The Quarkus Maven plugin is core-team-maintained; the Gradle plugin is community-driven with a smaller contributor base (https://quarkus.io/guides/gradle-tooling).

Maven Wrapper pins the exact build tool version per repository, eliminating version drift across developer machines, CI/CD runners, and production builds (https://maven.apache.org/wrapper/).

Maven's declarative `pom.xml` uses XML configuration only (https://maven.apache.org/pom.html), requiring no Groovy or Kotlin programming knowledge to read or modify.

# Implementation Guidelines:

## Mandatory Setup Steps:

1. **Maven Wrapper Verification and Creation**:
    - Verify Maven Wrapper presence before any development activities
    - **MUST** ensure all three wrapper components exist: `mvnw`, `mvnw.cmd`, and `.mvn/wrapper/maven-wrapper.jar`
    - **MUST NOT** commit `.gitignore` rules that exclude `maven-wrapper.jar` — the JAR is a required component
    - If wrapper missing, create using system Maven (temporary exception):
      ```bash
      # Create Maven Wrapper with specific version
      mvn -N wrapper:wrapper -Dmaven=3.9.8
      ```
    - If `maven-wrapper.jar` is missing (e.g., excluded by .gitignore), download it directly:
      ```bash
      curl -sL -o .mvn/wrapper/maven-wrapper.jar \
        https://repo.maven.apache.org/maven2/org/apache/maven/wrapper/maven-wrapper/3.2.0/maven-wrapper-3.2.0.jar
      ```
    - Commit wrapper files to version control:
      ```bash
      git add mvnw mvnw.cmd .mvn/wrapper/maven-wrapper.jar .mvn/wrapper/maven-wrapper.properties
      git commit -m "Add Maven Wrapper"
      ```

2. **Maven Wrapper Validation**:
   - Validate wrapper functionality after creation:
     ```bash
     ./mvnw -v
     ```
   - Verify output shows correct Maven version (3.9.8 or configured version)
   - Test basic build command:
     ```bash
     ./mvnw clean validate
     ```

3. **Environment Configuration**:
   - **MUST** use Maven Wrapper for all build commands
   - **SHOULD** configure IDE to use project's Maven Wrapper
   - **SHOULD** set IDE Maven runner to use wrapper script

4. **Build System Configuration**:
   - Configure Maven properties for Java version compatibility and centralized dependency versions:
     ```xml
     <properties>
         <maven.compiler.source>21</maven.compiler.source>
         <maven.compiler.target>21</maven.compiler.target>
         <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

         <!-- Dependency versions (single source of truth) -->
         <quarkus.platform.version>3.32.2</quarkus.platform.version>
         <clickhouse.version>0.6.3</clickhouse.version>
         <archunit.version>1.4.1</archunit.version>
         <!-- ... all other dependency versions ... -->
     </properties>
     ```
   - All `<dependency>` and `<plugin>` version references **MUST** use property placeholders:
     ```xml
     <dependency>
         <groupId>com.clickhouse</groupId>
         <artifactId>clickhouse-jdbc</artifactId>
         <version>${clickhouse.version}</version>
     </dependency>
     ```
   - **MUST NOT** hardcode version literals directly in dependency or plugin declarations.

5. **Version Management**:
   - Update Maven version by modifying `.mvn/wrapper/maven-wrapper.properties`:
     ```properties
     distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.8/apache-maven-3.9.8-bin.zip
     ```

## Validation Commands:
```bash
# Version check
./mvnw -v

# Basic build validation
./mvnw clean compile

# Dependency resolution test
./mvnw dependency:resolve
```

# Additional Recommendations:

## Build Optimization:
- **Maven Memory Configuration**: Configure Maven with appropriate heap settings for large projects:
  ```bash
  export MAVEN_OPTS="-Xmx2g -Xms1g -XX:+TieredCompilation -XX:TieredStopAtLevel=1"
  ```
- **Parallel Builds**: Enable parallel building for multi-module projects:
  ```bash
  ./mvnw clean install -T 4C  # 4 threads per CPU core
  ```

## Security and Quality:
- **Dependency Vulnerability Scanning**: Implement Maven dependency vulnerability scanning in build pipeline:
  ```xml
  <plugin>
      <groupId>org.owasp</groupId>
      <artifactId>dependency-check-maven</artifactId>
      <version>8.4.0</version>
  </plugin>
  ```
- **License Compliance**: Add license checking plugin for enterprise compliance
- **Code Quality**: Integrate quality gates with SonarQube or similar tools

## Development Productivity:
- **Maven Daemon**: Use Maven Daemon (mvnd) for faster builds in development:
  ```bash
  # Install mvnd as alternative to mvnw for local development only
  sdk install mvnd
  ```
- **IDE Integration**: Configure IDEs to use project Maven Wrapper automatically
- **Build Profiles**: Create development and production build profiles for optimized builds

## Automation and CI/CD:
- **Build Scripts**: Create standardized build scripts that wrap Maven Wrapper commands
- **Pipeline Integration**: Ensure CI/CD pipelines use Maven Wrapper exclusively
- **Artifact Publishing**: Configure Maven for artifact publishing to internal repositories

## Monitoring and Maintenance:
- **Version Updates**: Regularly update Maven Wrapper to latest stable versions
- **Build Metrics**: Collect and monitor build performance metrics
- **Dependency Updates**: Implement automated dependency update checks

## References:
- Apache Maven Official Documentation: https://maven.apache.org/guides/
- Maven Wrapper Documentation: https://maven.apache.org/wrapper/
- Quarkus Maven Tooling Guide: https://quarkus.io/guides/maven-tooling
- OWASP Dependency-Check Maven Plugin: https://jeremylong.github.io/DependencyCheck/dependency-check-maven/

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Maven Wrapper Files | `ls mvnw mvnw.cmd .mvn/wrapper/` | All files exist |
| Wrapper Executable | `./mvnw -v` | Returns Maven version 3.9.8 |
| Properties File | `cat .mvn/wrapper/maven-wrapper.properties` | Contains `apache-maven-3.9.8` |
| Clean Build | `./mvnw clean package -DskipTests` | Exit code 0 |
| Dependency Resolution | `./mvnw dependency:resolve` | Exit code 0 |
| Test Execution | `./mvnw test` | All tests pass |
| Checkstyle Pass | `./mvnw checkstyle:check` | No violations |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Maven Build Automation..."

# Check Maven Wrapper files exist (MUST include maven-wrapper.jar)
for file in mvnw mvnw.cmd .mvn/wrapper/maven-wrapper.properties .mvn/wrapper/maven-wrapper.jar; do
  if [[ ! -f "$file" ]]; then
    echo "❌ Missing: $file"
    exit 1
  fi
done
echo "✅ Maven Wrapper files present (including maven-wrapper.jar)"

# Verify Maven version
MAVEN_VERSION=$(./mvnw -v 2>/dev/null | grep 'Apache Maven' | awk '{print $3}')
if [[ "$MAVEN_VERSION" != "3.9.8" ]]; then
  echo "❌ Maven version $MAVEN_VERSION detected. Required: 3.9.8"
  exit 1
fi
echo "✅ Maven version: $MAVEN_VERSION"

# Test build
./mvnw clean compile -q
echo "✅ Build successful"

echo "✅ All Maven Build Automation criteria met"
```
