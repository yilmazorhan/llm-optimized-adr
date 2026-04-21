# Issue: Code Generation Templates for Implementation Consistency

Developers need concrete code templates to generate consistent implementations following architectural decisions. Without standardized templates, code generation produces implementations violating established patterns, naming conventions, and architectural boundaries.

# Decision:

Use standardized code generation templates for all common architectural patterns. All generated code **MUST** follow hexagonal architecture principles, reactive programming patterns, and established naming conventions while maintaining Quarkus and GraalVM native compilation compatibility.

**Required Template Categories**:
- Domain entities with JPA annotations and business validation
- Repository ports (interfaces) with reactive Mutiny return types  
- Application services with localized logging and exception handling
- REST controllers with OpenAPI documentation and DTO mapping
- Data Transfer Objects with Bean Validation integration

# Constraints:

- Generated code **MUST** compile with GraalVM native image
- Templates **MUST** support both blocking and reactive database operations
- Package structure **MUST** follow `com.copilot.quarkus.[layer].[domain].[component]` convention
- All generated classes **MUST** include proper OpenTelemetry tracing annotations
- Generated code **MUST** integrate with structured logging patterns

# Alternatives:

**IDE Templates**: IntelliJ IDEA Live Templates documentation (https://www.jetbrains.com/help/idea/using-live-templates.html) states templates are stored per-user in the IDE configuration directory as XML files, scoped to individual installations; exporting requires manual XML file sharing. Live templates produce text expansion only and cannot enforce structural rules such as hexagonal architecture layer separation, package naming conventions, or mandatory annotation patterns across generated files.

**External Code Generation Tools**: OpenAPI Generator (https://openapi-generator.tech/docs/customization/) uses Mustache/Handlebars templates that produce REST stubs from OpenAPI specifications. The OpenAPI Generator FAQ confirms it generates REST client/server code from API specs but does not support generating domain entity classes, hexagonal port interfaces, or application service layers. Custom template authoring requires maintaining a parallel template repository with Mustache syntax that has no integration with Quarkus CDI, Mutiny reactive types, or GraalVM native-image configuration.

# Rationale:

The Quarkus documentation (https://quarkus.io/guides/getting-started-reactive) requires specific patterns for reactive applications: `Uni` and `Multi` return types from SmallRye Mutiny, `@ApplicationScoped` CDI beans, and reactive Panache repository methods. The Jakarta Persistence 3.1 specification (https://jakarta.ee/specifications/persistence/3.1/) mandates `@Entity` classes with `@Table`, `@Id`, and `@GeneratedValue` annotations in prescribed combinations. Standardized templates encode these mandatory patterns once, eliminating per-implementation decisions. The GraalVM native-image documentation (https://www.graalvm.org/latest/reference-manual/native-image/metadata/) requires `@RegisterForReflection` on all entities and DTOs — a requirement templates embed automatically. Centralized template maintenance confines framework upgrade changes to template definitions rather than every generated class.

# Implementation Guidelines:

**Template Variable Reference**:
- `{context}`: Business domain context - lowercase (e.g., "user", "order")
- `{EntityName}`: PascalCase entity name (e.g., "User", "Order")  
- `{entity}`: camelCase entity name (e.g., "user", "order")
- `{entities}`: Plural camelCase (e.g., "users", "orders")
- `{ENTITY}`: Uppercase for constants (e.g., "USER", "ORDER")
- `{fieldName}`: camelCase field name (e.g., "firstName")
- `{FieldName}`: PascalCase field name (e.g., "FirstName")
- `{field}`: lowercase field name for validation keys
- `{field_name}`: snake_case for database columns

**Package Structure**: `com.copilot.quarkus.[layer].[domain].[component]`

## Template 1: Domain Entity (**CRITICAL**)

```java
package com.copilot.quarkus.domain.{context}.model;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import com.copilot.quarkus.infrastructure.i18n.MessageKeys;
import io.quarkus.runtime.annotations.RegisterForReflection;
import java.time.LocalDateTime;
import java.util.Objects;

/**
 * {EntityName} domain entity representing a {description} in the system.
 *
 * <p>This entity follows hexagonal architecture principles by containing only 
 * business logic and being free from infrastructure concerns. All validation
 * uses internationalized message keys for consistent error handling.
 */
@Entity
@Table(name = "{entities}")
@RegisterForReflection
public class {EntityName} {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "{field_name}", nullable = false, length = 255)
    @NotBlank(message = "{" + MessageKeys.Validation.{FIELD}_REQUIRED + "}")
    @Size(max = 255, message = "{" + MessageKeys.Validation.{FIELD}_SIZE + "}")
    private String {fieldName};

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at", nullable = false)  
    private LocalDateTime updatedAt;

    // JPA requires default constructor
    protected {EntityName}() {}

    // Business constructor with validation
    public {EntityName}(String {fieldName}) {
        this.{fieldName} = Objects.requireNonNull({fieldName}, "{FieldName} cannot be null");
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    // Factory method for creation
    public static {EntityName} create(String {fieldName}) {
        return new {EntityName}({fieldName});
    }

    // Business methods with automatic timestamp updates
    public void update{FieldName}(String new{FieldName}) {
        this.{fieldName} = Objects.requireNonNull(new{FieldName}, "New {fieldName} cannot be null");
        this.updatedAt = LocalDateTime.now();
    }

    // Standard getters
    public Long getId() { return id; }
    public String get{FieldName}() { return {fieldName}; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }

    // Entity equality based on business identity
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof {EntityName})) return false;
        {EntityName} {entity} = ({EntityName}) o;
        return Objects.equals(id, {entity}.id) && 
               Objects.equals({fieldName}, {entity}.{fieldName});
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, {fieldName});
    }

    @Override
    public String toString() {
        return "{EntityName}{id=" + id + ", {fieldName}='" + {fieldName} + "'}";
    }
}
```

## Template 2: Repository Port Interface (**ESSENTIAL**)

```java
package com.copilot.quarkus.domain.{context}.port;

import com.copilot.quarkus.domain.{context}.model.{EntityName};
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.Multi;

/**
 * Repository port for {EntityName} persistence operations.
 * 
 * <p>This port defines the contract for data access without specifying 
 * implementation details, following hexagonal architecture principles.
 * All operations return reactive types for non-blocking execution.
 */
public interface {EntityName}Repository {

    /**
     * Persists a new {entity} to the data store.
     * 
     * @param {entity} The {entity} to create
     * @return A Uni containing the persisted {entity} with generated ID
     */
    Uni<{EntityName}> save({EntityName} {entity});

    /**
     * Retrieves a {entity} by its unique identifier.
     * 
     * @param id The {entity} identifier
     * @return A Uni containing the {entity}, or null if not found
     */
    Uni<{EntityName}> findById(Long id);

    /**
     * Retrieves a {entity} by {fieldName}.
     * 
     * @param {fieldName} The {fieldName} to search for
     * @return A Uni containing the {entity}, or null if not found
     */
    Uni<{EntityName}> findBy{FieldName}(String {fieldName});

    /**
     * Retrieves all {entities} from the data store.
     * 
     * @return A Multi streaming all {entities}
     */
    Multi<{EntityName}> findAll();

    /**
     * Retrieves {entities} with pagination support.
     * 
     * @param page Page number (0-based)
     * @param size Number of items per page
     * @return A Multi streaming the requested page of {entities}
     */
    Multi<{EntityName}> findAll(int page, int size);

    /**
     * Updates an existing {entity} in the data store.
     * 
     * @param {entity} The {entity} to update
     * @return A Uni containing the updated {entity}
     */
    Uni<{EntityName}> update({EntityName} {entity});

    /**
     * Removes a {entity} from the data store.
     * 
     * @param id The identifier of the {entity} to delete
     * @return A Uni containing true if deletion was successful
     */
    Uni<Boolean> deleteById(Long id);

    /**
     * Checks if a {entity} exists by identifier.
     * 
     * @param id The {entity} identifier
     * @return A Uni containing true if the {entity} exists
     */
    Uni<Boolean> existsById(Long id);

    /**
     * Counts the total number of {entities}.
     * 
     * @return A Uni containing the total count
     */
    Uni<Long> count();
}
```

## Template 3: Application Service (**CRITICAL**)

```java
package com.copilot.quarkus.domain.{context}.service;

import com.copilot.quarkus.domain.{context}.model.{EntityName};
import com.copilot.quarkus.domain.{context}.port.{EntityName}Repository;
import com.copilot.quarkus.domain.{context}.exception.{EntityName}NotFoundException;
import com.copilot.quarkus.domain.{context}.exception.{EntityName}AlreadyExistsException;
import com.copilot.quarkus.infrastructure.logging.LocalizedLogger;
import com.copilot.quarkus.infrastructure.i18n.MessageKeys;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Application service for {EntityName} business operations.
 * 
 * <p>This service orchestrates domain operations and coordinates with infrastructure
 * adapters while maintaining business logic in the domain layer. All operations
 * include comprehensive logging and error handling.
 */
@ApplicationScoped
public class {EntityName}Service {

    private static final Logger logger = LoggerFactory.getLogger({EntityName}Service.class);

    @Inject
    {EntityName}Repository {entity}Repository;

    @Inject
    LocalizedLogger localizedLogger;

    /**
     * Creates a new {entity} with business validation.
     * 
     * @param {fieldName} The {entity} {fieldName}
     * @return A Uni containing the created {entity}
     */
    public Uni<{EntityName}> create{EntityName}(String {fieldName}) {
        localizedLogger.info(logger, MessageKeys.Log.{ENTITY}_CREATING, {fieldName});

        return {entity}Repository.findBy{FieldName}({fieldName})
            .onItem().ifNotNull().failWith(() -> {
                localizedLogger.warn(logger, MessageKeys.Log.{ENTITY}_VALIDATION_FAILED, 
                    "{FieldName} already exists: " + {fieldName});
                return new {EntityName}AlreadyExistsException({fieldName});
            })
            .onItem().ifNull().switchTo(() -> {
                {EntityName} new{EntityName} = {EntityName}.create({fieldName});
                
                return {entity}Repository.save(new{EntityName})
                    .invoke(saved{EntityName} -> {
                        localizedLogger.info(logger, MessageKeys.Log.{ENTITY}_CREATED, saved{EntityName}.getId());
                    });
            });
    }

    /**
     * Retrieves a {entity} by its unique identifier.
     * 
     * @param id The {entity} identifier
     * @return A Uni containing the {entity}
     * @throws {EntityName}NotFoundException if {entity} does not exist
     */
    public Uni<{EntityName}> findById(Long id) {
        localizedLogger.debug(logger, MessageKeys.Log.{ENTITY}_RETRIEVING_ID, id);

        return {entity}Repository.findById(id)
            .onItem().ifNull().failWith(() -> {
                localizedLogger.warn(logger, MessageKeys.Log.{ENTITY}_NOT_FOUND, id);
                return new {EntityName}NotFoundException(id);
            })
            .invoke({entity} -> {
                localizedLogger.debug(logger, "Successfully retrieved {entity} with ID: {}", {entity}.getId());
            });
    }

    /**
     * Retrieves all {entities} with optional pagination.
     * 
     * @return A Multi streaming all {entities}
     */
    public Multi<{EntityName}> findAll() {
        localizedLogger.debug(logger, MessageKeys.Log.{ENTITY}_RETRIEVING_ALL);
        
        return {entity}Repository.findAll()
            .invoke({entity} -> {
                localizedLogger.debug(logger, "Retrieved {entity}: {}", {entity}.getId());
            });
    }

    /**
     * Updates an existing {entity} with new information.
     * 
     * @param id The {entity} identifier
     * @param new{FieldName} The new {fieldName} value
     * @return A Uni containing the updated {entity}
     */
    public Uni<{EntityName}> update{EntityName}(Long id, String new{FieldName}) {
        return findById(id)
            .chain(existing{EntityName} -> {
                existing{EntityName}.update{FieldName}(new{FieldName});
                return {entity}Repository.update(existing{EntityName});
            })
            .invoke(updated{EntityName} -> {
                localizedLogger.info(logger, MessageKeys.Log.{ENTITY}_UPDATED, updated{EntityName}.getId());
            });
    }

    /**
     * Deletes a {entity} by its identifier.
     * 
     * @param id The {entity} identifier
     * @return A Uni containing true if deletion was successful
     */
    public Uni<Boolean> delete{EntityName}(Long id) {
        return findById(id)
            .chain(existing{EntityName} -> {entity}Repository.deleteById(id))
            .invoke(deleted -> {
                if (deleted) {
                    localizedLogger.info(logger, MessageKeys.Log.{ENTITY}_DELETED, id);
                } else {
                    localizedLogger.warn(logger, "Failed to delete {entity} with ID: {}", id);
                }
            });
    }

    /**
     * Counts the total number of {entities}.
     * 
     * @return A Uni containing the total count
     */
    public Uni<Long> count{EntityName}s() {
        return {entity}Repository.count();
    }
}
```

## Template 4: REST Controller (**MANDATORY**)

```java
package com.copilot.quarkus.infrastructure.web.controller;

import com.copilot.quarkus.domain.{context}.service.{EntityName}Service;
import com.copilot.quarkus.infrastructure.web.dto.Create{EntityName}Request;
import com.copilot.quarkus.infrastructure.web.dto.Update{EntityName}Request;
import com.copilot.quarkus.infrastructure.web.dto.{EntityName}Response;
import com.copilot.quarkus.infrastructure.web.dto.{EntityName}sResponse;
import com.copilot.quarkus.infrastructure.i18n.MessageKeys;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.Multi;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.validation.Valid;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.eclipse.microprofile.openapi.annotations.Operation;
import org.eclipse.microprofile.openapi.annotations.Parameter;
import org.eclipse.microprofile.openapi.annotations.responses.APIResponse;
import org.eclipse.microprofile.openapi.annotations.tags.Tag;
import org.eclipse.microprofile.openapi.annotations.media.Content;
import org.eclipse.microprofile.openapi.annotations.media.Schema;

/**
 * REST controller for {EntityName} management operations.
 * 
 * <p>This controller provides RESTful endpoints for {entity} CRUD operations
 * with proper security, validation, and API documentation.
 */
@Path("/api/v1/{entities}")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Tag(name = "{EntityName}s", description = "{EntityName} management operations") // MessageKeys.Api.TAG_{ENTITY}S_NAME, TAG_{ENTITY}S_DESCRIPTION
public class {EntityName}Controller {

    @Inject
    {EntityName}Service {entity}Service;

    @POST
    @Operation(
        summary = "Create a new {entity}", // MessageKeys.Api.{ENTITY}_CREATE_SUMMARY
        description = "Creates a new {entity} with the provided information") // MessageKeys.Api.{ENTITY}_CREATE_DESCRIPTION
    @APIResponse(
        responseCode = "201",
        description = "{EntityName} created successfully", // MessageKeys.Api.RESPONSE_201_DESCRIPTION
        content = @Content(schema = @Schema(implementation = {EntityName}Response.class)))
    @APIResponse(
        responseCode = "400", 
        description = "Invalid request data provided", // MessageKeys.Api.RESPONSE_400_DESCRIPTION
        content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @APIResponse(
        responseCode = "409",
        description = "{EntityName} already exists", // MessageKeys.Api.RESPONSE_409_DESCRIPTION
        content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @RolesAllowed({"ADMIN"})
    public Uni<Response> create{EntityName}(@Valid Create{EntityName}Request request) {
        return {entity}Service.create{EntityName}(request.get{FieldName}())
            .map({entity} -> Response.status(Response.Status.CREATED)
                .entity({EntityName}Response.from({entity}))
                .build());
    }

    @GET
    @Path("/{id}")
    @Operation(
        summary = "Retrieve {entity} by ID", // MessageKeys.Api.{ENTITY}_GET_SUMMARY  
        description = "Retrieves a {entity} by their unique identifier") // MessageKeys.Api.{ENTITY}_GET_DESCRIPTION
    @APIResponse(
        responseCode = "200",
        description = "{EntityName} retrieved successfully", // MessageKeys.Api.RESPONSE_200_DESCRIPTION
        content = @Content(schema = @Schema(implementation = {EntityName}Response.class)))
    @APIResponse(
        responseCode = "404",
        description = "{EntityName} not found", // MessageKeys.Api.RESPONSE_404_DESCRIPTION
        content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @RolesAllowed({"ADMIN", "USER"})
    public Uni<{EntityName}Response> get{EntityName}ById(
            @Parameter(description = "{EntityName} unique identifier") // MessageKeys.Api.PARAM_{ENTITY}_ID_DESCRIPTION
            @PathParam("id") Long id) {
        return {entity}Service.findById(id)
            .map({EntityName}Response::from);
    }

    @GET
    @Operation(
        summary = "Retrieve all {entities}", // MessageKeys.Api.{ENTITY}_GET_ALL_SUMMARY
        description = "Retrieves all {entities} with optional pagination") // MessageKeys.Api.{ENTITY}_GET_ALL_DESCRIPTION  
    @APIResponse(
        responseCode = "200",
        description = "{EntityName}s retrieved successfully", // MessageKeys.Api.RESPONSE_200_DESCRIPTION
        content = @Content(schema = @Schema(implementation = {EntityName}sResponse.class)))
    @RolesAllowed({"ADMIN"})
    public Uni<{EntityName}sResponse> getAll{EntityName}s() {
        return {entity}Service.findAll()
            .collect().asList()
            .map({EntityName}sResponse::from);
    }

    @PUT
    @Path("/{id}")
    @Operation(
        summary = "Update existing {entity}", // MessageKeys.Api.{ENTITY}_UPDATE_SUMMARY
        description = "Updates an existing {entity} with new information") // MessageKeys.Api.{ENTITY}_UPDATE_DESCRIPTION
    @APIResponse(
        responseCode = "200",
        description = "{EntityName} updated successfully", // MessageKeys.Api.RESPONSE_200_DESCRIPTION
        content = @Content(schema = @Schema(implementation = {EntityName}Response.class)))
    @APIResponse(
        responseCode = "404",
        description = "{EntityName} not found", // MessageKeys.Api.RESPONSE_404_DESCRIPTION  
        content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @RolesAllowed({"ADMIN"})
    public Uni<{EntityName}Response> update{EntityName}(
            @PathParam("id") Long id,
            @Valid Update{EntityName}Request request) {
        return {entity}Service.update{EntityName}(id, request.get{FieldName}())
            .map({EntityName}Response::from);
    }

    @DELETE
    @Path("/{id}")
    @Operation(
        summary = "Delete {entity} account", // MessageKeys.Api.{ENTITY}_DELETE_SUMMARY
        description = "Removes a {entity} from the system") // MessageKeys.Api.{ENTITY}_DELETE_DESCRIPTION
    @APIResponse(
        responseCode = "204",
        description = "{EntityName} deleted successfully") // MessageKeys.Api.RESPONSE_204_DESCRIPTION
    @APIResponse(
        responseCode = "404",
        description = "{EntityName} not found", // MessageKeys.Api.RESPONSE_404_DESCRIPTION
        content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @RolesAllowed({"ADMIN"})
    public Uni<Response> delete{EntityName}(@PathParam("id") Long id) {
        return {entity}Service.delete{EntityName}(id)
            .map(deleted -> Response.noContent().build());
    }
}
```

## Template 5: Data Transfer Objects (**REQUIRED**)

**Create Request DTO**:
```java
package com.copilot.quarkus.infrastructure.web.dto;

import com.copilot.quarkus.infrastructure.i18n.MessageKeys;
import com.fasterxml.jackson.annotation.JsonProperty;
import io.quarkus.runtime.annotations.RegisterForReflection;
import jakarta.validation.constraints.*;

/**
 * Request DTO for creating a new {entity}.
 * 
 * <p>Contains all required fields for {entity} creation with proper validation
 * annotations using internationalized message keys.
 */
@RegisterForReflection
public class Create{EntityName}Request {

    @JsonProperty("{fieldName}")
    @NotBlank(message = "{" + MessageKeys.Validation.{FIELD}_REQUIRED + "}")
    @Size(max = 255, message = "{" + MessageKeys.Validation.{FIELD}_SIZE + "}")
    private String {fieldName};

    // Default constructor for Jackson
    public Create{EntityName}Request() {}

    public Create{EntityName}Request(String {fieldName}) {
        this.{fieldName} = {fieldName};
    }

    // Getters and setters
    public String get{FieldName}() { return {fieldName}; }
    public void set{FieldName}(String {fieldName}) { this.{fieldName} = {fieldName}; }

    @Override
    public String toString() {
        return "Create{EntityName}Request{{fieldName}='" + {fieldName} + "'}";
    }
}
```

**Response DTO**:
```java
package com.copilot.quarkus.infrastructure.web.dto;

import com.copilot.quarkus.domain.{context}.model.{EntityName};
import com.fasterxml.jackson.annotation.JsonProperty;
import io.quarkus.runtime.annotations.RegisterForReflection;
import java.time.LocalDateTime;

/**
 * Response DTO for {EntityName} data.
 * 
 * <p>Provides a clean API representation of {entity} data without
 * exposing internal domain model structure.
 */
@RegisterForReflection
public class {EntityName}Response {

    @JsonProperty("id")
    private Long id;

    @JsonProperty("{fieldName}")
    private String {fieldName};

    @JsonProperty("createdAt")
    private LocalDateTime createdAt;

    @JsonProperty("updatedAt")
    private LocalDateTime updatedAt;

    // Default constructor for Jackson
    public {EntityName}Response() {}

    public {EntityName}Response(Long id, String {fieldName}, LocalDateTime createdAt, LocalDateTime updatedAt) {
        this.id = id;
        this.{fieldName} = {fieldName};
        this.createdAt = createdAt;
        this.updatedAt = updatedAt;
    }

    // Factory method for domain model conversion
    public static {EntityName}Response from({EntityName} {entity}) {
        return new {EntityName}Response(
            {entity}.getId(),
            {entity}.get{FieldName}(),
            {entity}.getCreatedAt(),
            {entity}.getUpdatedAt()
        );
    }

    // Getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String get{FieldName}() { return {fieldName}; }
    public void set{FieldName}(String {fieldName}) { this.{fieldName} = {fieldName}; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }

    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }

    @Override
    public String toString() {
        return "{EntityName}Response{id=" + id + ", {fieldName}='" + {fieldName} + "'}";
    }
}
```

## Mandatory Usage Rules (**SHALL** be followed):

1. **Template Completeness**: **MUST** replace all template variables before code compilation
2. **Package Structure**: **MUST** follow established hexagonal architecture package organization  
3. **Validation Integration**: **MUST** use `MessageKeys` constants for all validation messages
4. **Logging Requirements**: **MUST** include `LocalizedLogger` for all business operations
5. **Exception Handling**: **MUST** map domain exceptions to appropriate HTTP responses
6. **Reactive Patterns**: **MUST** use Mutiny `Uni` and `Multi` for all async operations
7. **Security Integration**: **MUST** include appropriate `@RolesAllowed` annotations
8. **API Documentation**: **MUST** include OpenAPI annotations with message key comments

## Validation Commands (**SHALL** be executed after generation):

```bash
# Verify generated code compiles successfully
./mvnw clean compile -DskipTests

# Run ArchUnit tests to validate architecture compliance
./mvnw test -Dtest=ArchitectureTest

# Verify message keys exist in resource bundles
./mvnw test -Dtest=MessageKeysValidationTest

# Test generated endpoints with sample data
curl -X POST http://localhost:8080/api/v1/{entities} \
     -H "Content-Type: application/json" \
     -d '{{fieldName}": "test-value"}'

# Verify OpenAPI documentation includes generated endpoints
curl http://localhost:8080/q/openapi | grep -i {entity}
```

# Additional Recommendations:

- Cache compiled templates for repeated generation to minimize I/O operations
- Provide IDE live templates for common variable patterns and snippets
- Update templates when framework versions change and validate Jakarta EE compatibility
- Ensure generated code passes ArchUnit architecture tests
- Do not commit generated code to version control

**References**:
- Quarkus Getting Started Reactive Guide: https://quarkus.io/guides/getting-started-reactive
- SmallRye Mutiny Documentation: https://smallrye.io/smallrye-mutiny/latest/
- Jakarta Persistence 3.1 Specification: https://jakarta.ee/specifications/persistence/3.1/
- GraalVM Native Image Metadata: https://www.graalvm.org/latest/reference-manual/native-image/metadata/
- MicroProfile OpenAPI 3.1 Specification: https://download.eclipse.org/microprofile/microprofile-open-api-3.1/
- ArchUnit User Guide: https://www.archunit.org/userguide/html/000_Index.html

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Template Directory | `find . -type d \( -name "templates" -o -name ".templates" -o -name "codegen" \) \| head -5` | At least one template directory exists |
| Entity Template | `find . -path "*/templates/*" -type f \| xargs grep -l "@Entity" 2>/dev/null \| head -5` | Entity generation template present |
| Repository Port Template | `find . -path "*/templates/*" -type f \| xargs grep -l "Repository" 2>/dev/null \| head -5` | Repository port template present |
| Service Template | `find . -path "*/templates/*" -type f \| xargs grep -l "@ApplicationScoped" 2>/dev/null \| head -5` | Service layer template present |
| REST Controller Template | `find . -path "*/templates/*" -type f \| xargs grep -l "@Path" 2>/dev/null \| head -5` | REST controller template present |
| DTO Template | `find . -path "*/templates/*" -type f \| xargs grep -l "Request\|Response" 2>/dev/null \| head -5` | DTO generation template present |
| Template Variables | `grep -rl "{EntityName}\|{context}\|{entity}" . --include="*.java" --include="*.ftl" --include="*.mustache" --include="*.qute" \| head -5` | Placeholder variables defined |
| Generated Code Compiles | `./mvnw clean compile -DskipTests` | Exit code 0 |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
echo "Validating Code Generation Templates..."

FAIL=0

# Detect template directory
TEMPLATE_DIR=""
for dir in templates .templates src/main/resources/templates codegen; do
  if [ -d "$dir" ]; then
    TEMPLATE_DIR="$dir"
    break
  fi
done

if [ -n "$TEMPLATE_DIR" ]; then
  echo "✅ Template directory found: $TEMPLATE_DIR"
else
  echo "❌ No template directory found (expected templates/, .templates/, src/main/resources/templates/, or codegen/)"
  FAIL=1
fi

# Check for entity template
FOUND=0
if [ -n "$TEMPLATE_DIR" ]; then
  for f in $(find "$TEMPLATE_DIR" -type f 2>/dev/null); do
    if grep -ql "@Entity\|{EntityName}" "$f" 2>/dev/null; then
      FOUND=1
      break
    fi
  done
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Entity generation template found"
else
  echo "❌ No entity generation template detected"
  FAIL=1
fi

# Check for repository port template
FOUND=0
if [ -n "$TEMPLATE_DIR" ]; then
  for f in $(find "$TEMPLATE_DIR" -type f 2>/dev/null); do
    if grep -ql "Repository\|RepositoryPort" "$f" 2>/dev/null; then
      FOUND=1
      break
    fi
  done
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Repository port template found"
else
  echo "❌ No repository port template detected"
  FAIL=1
fi

# Check for service template
FOUND=0
if [ -n "$TEMPLATE_DIR" ]; then
  for f in $(find "$TEMPLATE_DIR" -type f 2>/dev/null); do
    if grep -ql "@ApplicationScoped\|Service" "$f" 2>/dev/null; then
      FOUND=1
      break
    fi
  done
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Service layer template found"
else
  echo "❌ No service layer template detected"
  FAIL=1
fi

# Check for REST controller template
FOUND=0
if [ -n "$TEMPLATE_DIR" ]; then
  for f in $(find "$TEMPLATE_DIR" -type f 2>/dev/null); do
    if grep -ql "@Path\|Controller" "$f" 2>/dev/null; then
      FOUND=1
      break
    fi
  done
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ REST controller template found"
else
  echo "❌ No REST controller template detected"
  FAIL=1
fi

# Check for DTO template
FOUND=0
if [ -n "$TEMPLATE_DIR" ]; then
  for f in $(find "$TEMPLATE_DIR" -type f 2>/dev/null); do
    if grep -ql "Request\|Response\|Dto" "$f" 2>/dev/null; then
      FOUND=1
      break
    fi
  done
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ DTO generation template found"
else
  echo "❌ No DTO generation template detected"
  FAIL=1
fi

# Check for template variable placeholders
FOUND=0
if [ -n "$TEMPLATE_DIR" ]; then
  for f in $(find "$TEMPLATE_DIR" -type f 2>/dev/null); do
    if grep -ql "{EntityName}\|{context}\|{entity}" "$f" 2>/dev/null; then
      FOUND=1
      break
    fi
  done
fi
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Template variable placeholders defined"
else
  echo "❌ No template variable placeholders detected"
  FAIL=1
fi

# Verify generated code compiles
if ./mvnw clean compile -DskipTests -q 2>/dev/null; then
  echo "✅ Generated code compiles successfully"
else
  echo "❌ Compilation failed"
  FAIL=1
fi

# Run full build verification
if ./mvnw clean verify -q 2>/dev/null; then
  echo "✅ Full build verification passed"
else
  echo "❌ Build verification failed"
  FAIL=1
fi

if [ "$FAIL" -ne 0 ]; then
  echo "❌ Code Generation Templates validation failed"
  exit 1
fi

echo "✅ All Code Generation Templates criteria met"
```
