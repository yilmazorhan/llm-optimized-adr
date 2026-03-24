# Issue: Package Structure Organization for Hexagonal Architecture

Standardized package structure organization required to maintain hexagonal architecture boundaries and enable consistent code generation. Without explicit conventions, code violates architectural principles and creates maintenance complexity.

**Impact**: Inconsistent structure leads to boundary violations, difficulty locating components, and reduced development productivity.

# Decision:

Implement standardized package structure following hexagonal architecture with domain-driven design bounded contexts.

**Required Components**:
- Domain-driven hexagonal package organization **MUST** be consistently applied across all modules
- Standardized class and package naming patterns **MUST** be enforced
- Architectural boundaries **MUST** respect hexagonal dependency direction (inward only: infrastructure → application → domain)
- ArchUnit automated testing **MUST** prevent boundary violations at build time

# Constraints:

- Package dependencies **MUST** respect hexagonal architecture rules: infrastructure may depend on application and domain; application may depend on domain; domain **MUST NOT** depend on application or infrastructure.
- All bounded contexts **MUST** follow an identical package structure.
- ArchUnit tests **MUST** be present and **MUST** fail the build on any package boundary violation.
- Package names **MUST** be lowercase with no underscores.
- Bounded contexts **MUST** be isolated through package boundaries; cross-context access **MUST** go through port interfaces.
- **MUST NOT** place domain model classes in infrastructure or application packages.
- **MUST NOT** place adapter implementations in domain packages.

# Alternatives:

1. **Feature-based Packaging** (e.g., `com.example.user.UserController`, `com.example.user.UserRepository`, `com.example.user.User`): All classes for a feature grouped in one package regardless of architectural layer. Rejected because: ArchUnit's `noClasses().that().resideInAPackage()` rules cannot distinguish layers within a single feature package — there are no separate `domain`, `port`, or `adapter` packages to write rules against (https://www.archunit.org/userguide/html/000_Index.html#_package_dependency_checks). A REST controller and a repository adapter in the same package can freely import each other, making hexagonal boundary enforcement impossible at the package level.

2. **Technical Layer Packaging** (e.g., `com.example.controller.*`, `com.example.service.*`, `com.example.repository.*`): All controllers in one package, all services in another. Rejected because: This structure groups unrelated bounded contexts together — `UserController` and `OrderController` share the same `controller` package, meaning ArchUnit cannot enforce that `Order` domain code is isolated from `User` domain code. The Maven multi-module approach in ADR-09 requires per-context isolation which flat layer packages cannot provide.

# Rationale:

Hexagonal package structure maps directly to ArchUnit dependency rules: `noClasses().that().resideInAPackage("..domain..").should().dependOnClassesThat().resideInAnyPackage("..infrastructure..", "..application..")` — this rule is only enforceable when domain and infrastructure are separate packages (https://www.archunit.org/userguide/html/000_Index.html#_package_dependency_checks). The `{bounded-context}` sub-package level (e.g., `domain.user`, `domain.order`) enables per-context ArchUnit rules, preventing cross-context coupling. Consistent naming conventions (`*Port.java`, `*Adapter.java`, `*Service.java`) allow ArchUnit to verify that port packages contain only interfaces via `classes().that().resideInAPackage("..port..").should().beInterfaces()`. The predictable structure also enables code generation tools and LLM agents to place generated classes in the correct package without ambiguity.

# Implementation Guidelines:

## Package Structure:

```
com.example.
├── domain/
│   └── {bounded-context}/          # e.g., user, order, product
│       ├── model/                  # Domain entities and value objects
│       ├── port/                   # Inbound and outbound ports (interfaces)
│       ├── service/                # Domain services (business logic)
│       └── exception/              # Domain-specific exceptions
├── application/
│   └── {bounded-context}/
│       └── service/                # Application services (use cases)
└── infrastructure/
    ├── {bounded-context}/
    │   └── adapter/                # Outbound adapters (repositories, clients)
    ├── web/
    │   ├── controller/             # REST controllers (inbound adapters)
    │   ├── dto/                    # Data transfer objects
    │   └── exception/              # HTTP exception handlers
    ├── config/                     # Configuration classes
    ├── security/                   # Security implementations
    ├── i18n/                      # Internationalization services
    └── validation/                 # Validation configuration
```

## Dependency Rules (ArchUnit Enforced):
- Domain packages: Cannot import from `application` or `infrastructure`
- Application packages: May import from `domain`, cannot import from `infrastructure`  
- Infrastructure packages: May import from `domain` and `application`

## Jandex Indexing (Required for Multi-Module CDI Discovery):
All library modules (non-application modules) **MUST** include the `jandex-maven-plugin` in their `pom.xml`. Quarkus only discovers CDI beans, JAX-RS resources, and other annotated classes in dependency JARs when a Jandex index (`META-INF/jandex.idx`) is present. Without this, classes such as `@Path` REST resources, `@ApplicationScoped` beans, and `@Provider` exception mappers in library modules will **not** be discovered at runtime, resulting in 404 errors or missing injection points.

```xml
<plugin>
    <groupId>io.smallrye</groupId>
    <artifactId>jandex-maven-plugin</artifactId>
    <version>3.5.3</version>
    <executions>
        <execution>
            <id>make-index</id>
            <goals>
                <goal>jandex</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

**Applies to**: `exception-library`, `i18n-library`, `platform-api`, `platform-infrastructure` — every module except the final `container-app` (which is the Quarkus application entry point and is indexed automatically).

## Naming Conventions:
- **Entities**: `User.java`, `Order.java` (singular, PascalCase)
- **Ports**: `UserRepositoryPort.java`, `NotificationPort.java` (interface with "Port" suffix)
- **Adapters**: `ClickHouseUserRepositoryAdapter.java`, `EmailNotificationAdapter.java` (implementation with "Adapter" suffix)
- **Services**: `UserService.java`, `UserApplicationService.java` (context + "Service")
- **Controllers**: `UserController.java`, `OrderController.java` (entity + "Controller")
- **DTOs**: `CreateUserRequest.java`, `UserResponse.java` (operation + "Request"/"Response")
- **Exceptions**: `UserNotFoundException.java`, `InvalidEmailException.java` (descriptive + "Exception")

## Code Generation Guidelines:
1. Domain entities: `com.example.domain.{context}.model`
2. Repository ports: `com.example.domain.{context}.port`
3. REST controllers: `com.example.infrastructure.web.controller`
4. Consistent naming: Entity + Port/Adapter/Service/Controller suffix
5. Dependency compliance: Never import infrastructure packages in domain

# Additional Recommendations:

- **RECOMMENDED** Integrate ArchUnit package dependency validation in CI/CD with build failure on violations.
- **RECOMMENDED** Use IDE templates for auto-generating correct package structures.
- **RECOMMENDED** Add `package-info.java` files with architectural context documentation.
- **RECOMMENDED** Train team on package patterns and provide refactoring tools for migration.
- **RECOMMENDED** Monitor package metrics for coupling and cohesion with `./mvnw dependency:analyze`.
- **MUST** Reference ADR-09 (Multi-Module Structure), ADR-10 (Hexagonal Architecture), ADR-08 (Quarkus Framework).
- See ArchUnit Package Dependency Checks: https://www.archunit.org/userguide/html/000_Index.html#_package_dependency_checks
- See Quarkus CDI Reference (for port/adapter injection): https://quarkus.io/guides/cdi-reference
- See Hexagonal Architecture: https://alistair.cockburn.us/hexagonal-architecture/

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Package Convention | Directory structure | Matches `com.organization.project.module.layer` |
| Domain Package | `find . -path "*domain/model*"` | Model classes exist |
| Application Package | `find . -path "*application/service*"` | Service classes exist |
| Infrastructure Package | `find . -path "*infrastructure/adapter*"` | Adapters exist |
| No Cyclic Dependencies | ArchUnit test | No package cycles |
| Package Naming Rules | ArchUnit validation | Lowercase, no underscores |
| Build Success | `./mvnw clean compile` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Package Structure Organization..."

# Check for required package layers
REQUIRED_PACKAGES=("domain" "application" "infrastructure")
for pkg in "${REQUIRED_PACKAGES[@]}"; do
  if find . -type d -name "$pkg" 2>/dev/null | grep -q .; then
    echo "✅ Package layer '$pkg' exists"
  else
    echo "❌ Package layer '$pkg' not found"
    exit 1
  fi
done

# Check for domain subpackages
for sub in model port service; do
  if find . -path "*domain*$sub*" -type d 2>/dev/null | grep -q .; then
    echo "✅ Domain subpackage '$sub' exists"
  else
    echo "❌ Domain subpackage '$sub' not found"
    exit 1
  fi
done

# Check for lowercase package names (no uppercase in directory names under src)
UPPERCASE_PKG=$(find ./*/src -type d -name "*[A-Z]*" 2>/dev/null | head -5)
if [ -z "$UPPERCASE_PKG" ]; then
  echo "✅ All package names are lowercase"
else
  echo "❌ Found uppercase in package names: $UPPERCASE_PKG"
  exit 1
fi

# Build to verify package structure compiles
./mvnw clean compile -q && echo "✅ Package structure compiles successfully"

echo "✅ All Package Structure Organization criteria met"
```
