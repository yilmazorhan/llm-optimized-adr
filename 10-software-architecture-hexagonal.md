# Issue: Software Architecture Pattern Selection

Software architecture pattern required to ensure long-term maintainability, testability, and adaptability for cloud-native microservices. Without proper boundaries, applications suffer from tight coupling, difficult testing, and inability to adapt to changing requirements.

**Impact**: Poor architecture leads to increased development costs, reduced agility, and difficulty maintaining code quality as the system evolves.

# Decision:

Implement Hexagonal Architecture (Ports and Adapters pattern) with automated boundary enforcement using ArchUnit for all microservices.

**Required Components**:
- Hexagonal (Ports and Adapters) **MUST** be consistently applied across all modules
- ArchUnit automated testing **MUST** prevent architectural violations at build time
- Domain, Ports, Adapters layer separation **MUST** maintain inward dependency direction
- Automated architectural tests **MUST** run in CI/CD pipeline

# Constraints:

- Domain layer **MUST NOT** depend on adapter or infrastructure packages.
- Ports **MUST** be defined as Java interfaces only; **MUST NOT** contain implementation logic.
- Adapters **MUST** implement port interfaces; **MUST NOT** be referenced by domain code.
- Dependency direction **MUST** point inward: Adapters → Ports → Domain. The reverse **MUST NOT** occur.
- ArchUnit tests **MUST** be present and **MUST** fail the build on any layer violation.
- Abstraction layer overhead **MUST NOT** add >15ms latency per request.
- **MUST NOT** add >20MB additional memory footprint for port/adapter interfaces.

# Alternatives:

1. **Layered Architecture (Controller → Service → Repository)**: Traditional three-tier pattern where each layer can call the layer below it. Rejected because: Layered architecture permits the Service layer to directly depend on Repository implementation classes, not interfaces. This means replacing a database (e.g., PostgreSQL → ClickHouse) requires changes in both Service and Repository layers. ArchUnit's `layeredArchitecture()` API documents this limitation — layers define access rules but not dependency inversion (https://www.archunit.org/userguide/html/000_Index.html#_layer_checks). Without enforced interfaces, unit testing Service classes requires the real Repository or framework-specific mocking.

2. **Clean Architecture (Uncle Bob)**: Defines 4 concentric rings (Entities, Use Cases, Interface Adapters, Frameworks). Rejected because: Clean Architecture prescribes separate Entity and Use Case layers that produce 4 packages per bounded context versus Hexagonal's 3 (Domain, Port, Adapter). For microservices with a single bounded context, the Entity/Use Case split adds an extra layer boundary without additional isolation benefit. The original Clean Architecture article (https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) targets monolithic applications where multiple use cases share entities across bounded contexts — a scenario not applicable to single-responsibility microservices.

# Rationale:

Hexagonal Architecture was defined by Alistair Cockburn (https://alistair.cockburn.us/hexagonal-architecture/) to enforce that application logic has no dependency on external systems. In this pattern, the domain layer depends only on port interfaces; adapters implement those interfaces and depend inward. This means ArchUnit can verify at build time that `..domain..` packages have zero imports from `..adapter..` packages (demonstrated in the ArchUnit user guide: https://www.archunit.org/userguide/html/000_Index.html#_layer_checks). Replacing an external system (e.g., swapping a ClickHouse adapter for an in-memory test adapter) requires only a new adapter class implementing the same port interface — domain code remains unchanged. Quarkus CDI (`@ApplicationScoped`, `@Inject`) natively supports this pattern by injecting adapter implementations into domain classes via port interfaces (https://quarkus.io/guides/cdi-reference).

# Implementation Guidelines:

## ArchUnit Integration:

**Maven Dependency:**
```xml
<dependency>
  <groupId>com.tngtech.archunit</groupId>
  <artifactId>archunit-junit5-api</artifactId>
  <version>1.4.1</version>
  <scope>test</scope>
</dependency>
```

## Architectural Test Implementation:

```java
import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.lang.syntax.ArchRuleDefinition;
import org.junit.jupiter.api.Test;

class HexagonalArchitectureTest {
    private final JavaClasses importedClasses = new ClassFileImporter().importPackages("com.example");

    @Test
    void domainShouldNotDependOnAdaptersOrInfrastructure() {
        ArchRuleDefinition.noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..adapter..", "..infrastructure..")
            .check(importedClasses);
    }

    @Test
    void adaptersShouldOnlyDependOnAllowedPackages() {
        ArchRuleDefinition.classes()
            .that().resideInAPackage("..adapter..")
            .and().areNotAnnotatedWith(org.junit.jupiter.api.Test.class)
            .and().haveSimpleNameNotEndingWith("Test")
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage(
                "..adapter..",
                "..port..",
                "..domain..",
                "..infrastructure..",
                "java..",
                "jakarta..",
                "org.hibernate..",
                "org.slf4j..",
                "io.smallrye..",
                "io.quarkus.."
            )
            .check(importedClasses);
    }

    @Test
    void portsShouldNotDependOnAdapters() {
        ArchRuleDefinition.noClasses()
            .that().resideInAPackage("..port..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..adapter..")
            .check(importedClasses);
    }
}
```

## Package Structure:
- Organize code using hexagonal package structure
- Follow domain-driven design principles for package naming
- Maintain clear separation between domain, ports, and adapters

## Build Integration:
- Integrate architectural tests into CI/CD pipeline
- Configure build to fail on architectural violations
- Run tests with every build: `./mvnw test`

# Additional Recommendations:

- **RECOMMENDED** Combine hexagonal with event-driven patterns at service boundaries for scalability.
- **RECOMMENDED** Keep ports focused (<7 methods per interface) and design adapters for non-blocking reactive operations.
- **RECOMMENDED** Implement integration tests for each adapter using mock port implementations.
- **RECOMMENDED** Configure IDE plugins (e.g., IntelliJ Architecture plugin) to highlight violations.
- **RECOMMENDED** Monitor dependency graphs with `./mvnw dependency:tree` to identify drift.
- **MUST** Reference ADR-09 (Multi-Module Structure), ADR-11 (Package Structure), ADR-08 (Quarkus Framework).
- See Hexagonal Architecture original definition: https://alistair.cockburn.us/hexagonal-architecture/
- See ArchUnit User Guide: https://www.archunit.org/userguide/html/000_Index.html
- See Quarkus CDI Reference: https://quarkus.io/guides/cdi-reference

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| ArchUnit Dependency | `grep archunit pom.xml` | archunit-junit5-api present |
| Domain Independence | ArchUnit test | Domain has no adapter imports |
| Port Interfaces | Code inspection | Ports are interfaces only |
| Adapter Implementations | Code inspection | Adapters implement ports |
| Hexagonal Tests Pass | `./mvnw test -Dtest=*Hexagonal*` | All tests pass |
| No Layer Violations | ArchUnit validation | Domain→App→Infra direction only |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Hexagonal Architecture..."

# Check ArchUnit dependency
if grep -rq "archunit" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ ArchUnit dependency found"
else
  echo "❌ ArchUnit dependency not found in pom.xml"
  exit 1
fi

# Check for hexagonal package structure
for pkg in domain port adapter; do
  if find . -type d -name "$pkg" | grep -q .; then
    echo "✅ Package '$pkg' exists"
  else
    echo "❌ Package '$pkg' not found"
    exit 1
  fi
done

# Run architecture tests (MUST fail build on violations)
./mvnw test -Dtest="*Architecture*,*Hexagonal*" -q
echo "✅ Architecture tests passed"

# Build and verify
./mvnw clean verify -DskipTests -q
echo "✅ Build verification successful"

echo "✅ All Hexagonal Architecture criteria met"
```
