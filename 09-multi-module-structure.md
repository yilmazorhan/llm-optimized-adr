# Issue: Multi-Module Maven Architecture

Enterprise Java applications need scalable, maintainable multi-module architecture with clear separation of concerns and no circular dependencies.

**Impact**: Poor structure causes tight coupling, difficult maintenance, challenging testing, and reduced team scalability.

# Decision:

Implement multi-module Maven architecture with distinct module types that enforce strict separation of concerns. Communication between layers **MUST** occur via interfaces. Libraries **MUST** have a single responsibility.

## Module Types & Responsibilities

| Module Type | Responsibility | Constraints |
| :--- | :--- | :--- |
| **Business System** | Core domain logic, rules, use cases. | **MUST NOT** depend on implementation details (DB, HTTP). **MUST** access external systems via interfaces. |
| **External System** | Infrastructure implementations (JDBC, ORM, 3rd-party APIs). | **MUST** expose interfaces for business consumption. **MUST NOT** contain business logic. |
| **Web API** | REST Controllers, OpenAPI, HTTP handling. | **MUST** depend only on Business interfaces. **MUST NOT** contain business logic or DB entities. |
| **Library** | Single technical or domain concern (e.g., `security-lib`, `user-dto-lib`). | **MUST** have **ONE** responsibility. **MUST NOT** mix concerns (e.g., HTTP + DB). |
| **Container** | Deployment assembly. | **MUST NOT** contain code. |

# Constraints:

- **No Circular Dependencies**: Modules **MUST** maintain a DAG (Directed Acyclic Graph) structure.
- **No Library Cycles**: Circular dependencies between libraries (e.g., `lib-a` ↔ `lib-b`) are **STRICTLY FORBIDDEN**. Shared code must be extracted to a third library (`lib-c`).
- **Interface-Based Communication**: Business modules **MUST** access external systems via interfaces only.
- **Single Responsibility Libraries**: Libraries **MUST** focus on a single concern.
- **No "God" Libraries**: A single `common-shared` library containing unrelated concerns is **FORBIDDEN**.
- **No Mixed Concerns**: Libraries **MUST NOT** depend on both `jakarta.ws.rs.*` (HTTP) and `jakarta.persistence.*` (DB).

# Alternatives:

1. **Monolithic Single Module**: All source code in one Maven module with package-level separation only. Rejected because: Maven `dependency:analyze` cannot detect cross-layer violations within a single module — there are no module boundaries to enforce. A single module produces one JAR, meaning all transitive dependencies (HTTP, DB, security) are included in every deployment artifact regardless of need. The Maven Enforcer `bannedDependencies` rule cannot restrict intra-module package access (https://maven.apache.org/enforcer/enforcer-rules/bannedDependencies.html).

2. **Single "common-shared" Library**: All cross-cutting concerns (exceptions, validation, security, i18n) in one shared library module. Rejected because: Any module depending on `common-shared` transitively pulls all of its dependencies. If `common-shared` contains both `jakarta.persistence.*` and `jakarta.ws.rs.*` classes, a Web API module gains a compile-time dependency on persistence libraries it does not use. This pattern is documented as the "God Object" anti-pattern in Maven multi-module projects (https://maven.apache.org/guides/mini/guide-multiple-modules.html).

# Rationale:

Maven multi-module projects enforce compile-time boundaries: a module can only access classes from explicitly declared `<dependency>` entries in its `pom.xml` (https://maven.apache.org/guides/mini/guide-multiple-modules.html). This means a Web API module physically cannot import a DB entity class unless it declares a dependency on the persistence module. ArchUnit tests can verify layer rules at the class level, failing the build on violations (https://www.archunit.org/userguide/html/000_Index.html#_layer_checks). The Maven Enforcer plugin's `bannedDependencies` rule prevents forbidden transitive dependencies from entering specific modules (https://maven.apache.org/enforcer/enforcer-rules/bannedDependencies.html). Single-responsibility libraries keep transitive dependency trees small — a module needing only `exception-library` does not inherit validation, security, or i18n dependencies.

# Implementation Guidelines:

## 1. Maven Structure Example

```xml
<modules>
    <!-- Business Logic (Pure Java) -->
    <module>order-business</module>

    <!-- Infrastructure (Impl) -->
    <module>order-persistence-clickhouse</module>

    <!-- API (HTTP) -->
    <module>order-web-api</module>

    <!-- Granular Libraries -->
    <module>shared-kernel-domain</module> <!-- Value Objects only -->
    <module>security-utils</module>       <!-- Security only -->
</modules>
```

## 2. Anti-Patterns (Strictly Forbidden)

> [!WARNING]
> The following patterns are explicitly **FORBIDDEN**.

### ❌ The "God" Shared Library
**Do NOT** create a single `common-shared` library containing everything.
*   *Bad*: A library containing `UserEntity` (DB), `UserResponse` (HTTP), and `StringUtil`.
*   *Why*: Forces the Web API to depend on Database libraries. Violates architectural boundaries.

### ❌ Mixed Concern Libraries
**Do NOT** mix layers in a single library.
*   *Bad*: A library with `jakarta.persistence.*` (DB) and `jakarta.ws.rs.*` (HTTP) dependencies.
*   *Correction*: Split into `user-data-lib` and `user-api-lib`.

## 3. Validation
*   **ArchUnit Tests**: MUST fail build if circular dependencies or layer violations are detected.
*   **Maven Enforcer**: MUST fail build if banned dependencies (e.g., DB libs in API module) are found.

## 4. Multi-Module Development Commands

When working with multi-module projects, the correct Maven flags **MUST** be used:

| Command | Purpose | Required Flags |
|---------|---------|----------------|
| Build all | Install all modules | `./mvnw clean install` |
| Build specific | Build module + deps | `./mvnw clean install -pl <module> -am` |
| Dev mode | Run with live reload | `./mvnw quarkus:dev -pl <container-module> -am` |
| Package | Package for deployment | `./mvnw clean package -pl <container-module> -am` |

**The `-am` (also-make) Flag**:
- **MUST** be used when running goals on a specific module that depends on other project modules
- Ensures dependent modules are built first before the target module
- Without `-am`, Maven **SHALL** fail to find sibling module dependencies if they're not in the local repository

**Example - Quarkus Dev Mode:**
```bash
# CORRECT: MUST use -am flag to build dependencies first
./mvnw quarkus:dev -pl container-app -am

# INCORRECT: SHALL fail if dependencies not installed
./mvnw quarkus:dev -pl container-app
```

**Makefile Pattern:**
```makefile
dev: ## Start development server with live reload
	# MUST include -am flag for multi-module projects
	./mvnw quarkus:dev -pl container-app -am

package: ## Package application
	# MUST include -am flag for multi-module projects
	./mvnw clean package -DskipTests -pl container-app -am
```

# Additional Recommendations:

**Library Creation Rules**:
- **RECOMMENDED** Create a library when 2+ modules need identical functionality.
- **RECOMMENDED** Extract to a new library when a circular dependency is detected.
- **RECOMMENDED** Name format: `<concern>-library` (e.g., `exception-library`, `validation-library`).
- **RECOMMENDED** Process: Analyze → Create → Integrate → Validate.
- **MUST** Reference ADR-08 (Quarkus Framework), ADR-10 (Hexagonal Architecture), ADR-11 (Package Structure).
- See Maven Multi-Module Guide: https://maven.apache.org/guides/mini/guide-multiple-modules.html
- See ArchUnit User Guide: https://www.archunit.org/userguide/html/000_Index.html
- See Maven Enforcer Plugin: https://maven.apache.org/enforcer/maven-enforcer-plugin/

---

## Platform Library Catalog

> [!IMPORTANT]
> This section defines the **ONLY** allowed platform-level libraries.
> Creating libraries outside this catalog requires architecture review.

## Approved Platform Libraries

| Library Name | Single Responsibility | Allowed Contents | Forbidden Contents |
|-------------|----------------------|------------------|-------------------|
| `exception-library` | Exception handling | `PlatformException`, domain exceptions, error codes | i18n, HTTP, DB, utilities |
| `i18n-library` | Internationalization | `MessageKeys`, `Messages`, locale utilities | Exceptions, HTTP, DB |
| `validation-library` | Validation utilities | Custom validators, validation helpers | Business logic, HTTP, DB |
| `security-library` | Security utilities | Encryption, hashing, token utilities | HTTP filters, business logic |
| `date-time-library` | Date/time utilities | Timezone handling, formatting | Business logic |

## Forbidden Library Names

> [!CAUTION]
> The following library names are **EXPLICITLY FORBIDDEN** as they encourage "God Library" anti-patterns:

- ❌ `shared-library`
- ❌ `common-library`
- ❌ `utils-library`
- ❌ `core-library`
- ❌ `base-library`
- ❌ `platform-shared`
- ❌ `common-utils`

**Why**: These names are ambiguous and naturally attract unrelated code over time, violating single responsibility.

---

## Pre-Implementation Checklist

> [!IMPORTANT]
> Before creating ANY new library module, this checklist **MUST** be completed.

## Library Creation Checklist

- [ ] **Single Concern Identified**: Can you describe the library's purpose in ONE sentence without using "and"?
- [ ] **Name Specificity**: Does the name clearly indicate the single concern? (e.g., `exception-library` not `shared-library`)
- [ ] **Forbidden Name Check**: Is the proposed name NOT in the forbidden list above?
- [ ] **Dependency Check**: Will this library require ONLY standard Java or framework-agnostic dependencies?
- [ ] **No Mixed Layers**: Will this library avoid mixing HTTP (`jakarta.ws.rs.*`) and DB (`jakarta.persistence.*`) concerns?
- [ ] **Catalog Compliance**: Is this library in the approved catalog OR has architecture review been requested?

## Code Review Gate

Pull requests introducing new library modules **MUST** include:
1. Completed checklist above
2. Justification if not in approved catalog
3. ArchUnit test verifying single responsibility

---

## Enforcement

## Automated Checks (CI/CD)

```java
// ArchUnit: Verify no "shared" or "common" library exists
@ArchTest
static final ArchRule no_god_libraries = noClasses()
    .that().resideInAPackage("..shared..")
    .or().resideInAPackage("..common..")
    .should().exist()
    .because("God libraries violate ADR-09 single responsibility rule");
```

## Maven Enforcer Rule (pom.xml)

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <executions>
        <execution>
            <id>ban-god-libraries</id>
            <goals><goal>enforce</goal></goals>
            <configuration>
                <rules>
                    <bannedDependencies>
                        <excludes>
                            <exclude>*:shared-library:*</exclude>
                            <exclude>*:common-library:*</exclude>
                            <exclude>*:utils-library:*</exclude>
                        </excludes>
                        <message>God libraries are forbidden per ADR-09</message>
                    </bannedDependencies>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Module Structure | `find . -name "pom.xml" -type f` | Business/External/API modules exist |
| No Circular Dependencies | `./mvnw dependency:analyze` | No circular refs |
| ArchUnit Tests | `./mvnw test -Dtest=*ArchTest*` | All pass |
| No God Libraries | `ls */pom.xml \| xargs grep -l "shared-library"` | No matches |
| Single Responsibility | Manual review | Each module one concern |
| Build Order | `./mvnw clean install` | Builds in correct order |
| Interface Separation | Code review | Business via interfaces only |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Multi-Module Structure..."

# Check for forbidden library names
FORBIDDEN="shared-library common-library utils-library core-library"
for name in $FORBIDDEN; do
  if find . -name "pom.xml" -exec grep -l "$name" {} \; 2>/dev/null | grep -q .; then
    echo "❌ Forbidden library found: $name"
    exit 1
  fi
done
echo "✅ No forbidden library names"

# Build all modules
./mvnw clean install -DskipTests -q
echo "✅ All modules build successfully"

# Run ArchUnit tests (MUST fail build on violations)
./mvnw test -Dtest="*Arch*,*Architecture*" -q
echo "✅ Architecture tests passed"

echo "✅ All Multi-Module Structure criteria met"
```
