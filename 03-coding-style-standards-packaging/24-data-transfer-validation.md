# Issue: Data Transfer Object Validation Patterns

Standardized DTO patterns, validation annotations, and data mapping needed for consistent API contracts across the application.

# Decision:

Implement standardized DTO patterns with comprehensive Bean Validation 3.0 integration. All DTOs **MUST** use internationalized message keys and **MUST** follow consistent naming conventions for API contract stability.

**Required Components**:
- **Request DTOs**: Immutable classes with comprehensive validation annotations
- **Response DTOs**: Java records for immutable data representation
- **Update DTOs**: Partial update patterns with null-safe handling
- **Validation Integration**: Bean Validation 3.0 with internationalized messages
- **Message Key Management**: Centralized message key constants for validation and API documentation

# Constraints:

- All validation annotations **MUST** be compatible with the Bean Validation 3.0 specification (Jakarta Validation).
- DTOs **MUST** support JSON serialization/deserialization without data loss or security exposures.
- All validation messages **MUST** use internationalized message keys; hardcoded strings **MUST NOT** be used.
- Update DTOs **MUST** handle null values safely without overwriting existing data unintentionally.
- Response DTOs **MUST NOT** expose internal implementation details, technical IDs, or sensitive business logic.
- DTO changes **SHOULD** maintain backward compatibility for external API consumers.
- JSON property names **MUST** follow camelCase convention and remain stable across versions.
- DTO mapping operations **SHOULD** minimize object allocation overhead in high-throughput scenarios.

# Alternatives:

### 1. Direct Domain Entity Exposure

Exposing JPA or domain entities directly as REST API request/response objects couples the API contract to the internal data model. When a database column is renamed or a domain field type changes, all API consumers break simultaneously. The Jakarta Persistence specification (https://jakarta.ee/specifications/persistence/3.1/) documents that entity lifecycle callbacks (`@PrePersist`, `@PreUpdate`) and lazy-loaded associations can trigger unexpected behavior during JSON serialization, including `LazyInitializationException` and circular reference loops. Quarkus RESTEasy Reactive serializes response objects on the I/O thread (https://quarkus.io/guides/resteasy-reactive#serialization), meaning entity proxy resolution could block the event loop.

### 2. Programmatic Validation (Manual Checks)

Replacing annotation-based Bean Validation with programmatic `if/else` checks in each endpoint method scatters validation logic across controllers. The Jakarta Bean Validation 3.0 specification (https://jakarta.ee/specifications/bean-validation/3.0/) provides declarative constraint annotations that are automatically invoked by the Quarkus RESTEasy Reactive integration before the endpoint method executes (https://quarkus.io/guides/validation). Manual validation requires each developer to remember and re-implement the same checks, producing inconsistent error messages and missing validations across endpoints.

# Rationale:

DTOs create a boundary between the internal domain model and the public API contract, allowing each to evolve independently. The Quarkus RESTEasy Reactive guide (https://quarkus.io/guides/resteasy-reactive) documents that request deserialization and response serialization operate on dedicated types, preventing domain entity lifecycle side effects during HTTP processing.

Bean Validation 3.0 annotations (`@NotBlank`, `@Size`, `@Email`, `@Pattern`) are processed automatically by the Quarkus Hibernate Validator integration (https://quarkus.io/guides/validation). Constraint violations produce structured error responses without manual validation code. The `@RegisterForReflection` annotation ensures DTO classes are available for JSON serialization under GraalVM native compilation (https://quarkus.io/guides/writing-native-applications-tips#registering-for-reflection).

Internationalized message keys (`{validation.email.required}`) resolve against resource bundles at runtime, enabling multi-locale error messages as defined in the Hibernate Validator documentation (https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#section-message-interpolation). Java records for response DTOs provide immutability and reduced boilerplate as specified in JEP 395 (https://openjdk.org/jeps/395).

# Implementation Guidelines:


**Mandatory DTO Implementation** (**MUST** be completed):

**Naming Conventions** (**MUST** follow):
- Request DTOs: `Create[EntityName]Request`, `Update[EntityName]Request` (e.g., `CreateUserRequest`, `UpdateUserRequest`)
- Response DTOs: `[EntityName]Response`, `[EntityName]ListResponse` (e.g., `UserResponse`, `UserListResponse`)
- Validation message keys: `validation.[field].[rule]` (e.g., `validation.email.required`, `validation.firstName.size`)
- API message keys: `api.[entity].[operation].[type]` (e.g., `api.user.create.summary`, `api.response.201.description`)
- Custom validators: `[BusinessRule]Validator` (e.g., `BusinessEmailValidator`, `UniqueEmailValidator`)

## 1. Request DTO Pattern

**Complete Create Request Template**:
```java
package com.copilot.quarkus.infrastructure.web.dto;

import com.copilot.quarkus.infrastructure.i18n.MessageKeys;
import jakarta.validation.constraints.*;
import com.fasterxml.jackson.annotation.JsonProperty;
import io.quarkus.runtime.annotations.RegisterForReflection;

/**
 * Request DTO for creating a new User.
 * 
 * <p>This DTO validates input data according to business rules and 
 * provides localized validation messages for comprehensive error handling.
 */
@RegisterForReflection // GraalVM native compilation support
public class CreateUserRequest {

    @JsonProperty("email")
    @NotBlank(message = "{" + MessageKeys.Validation.EMAIL_REQUIRED + "}")
    @Email(message = "{" + MessageKeys.Validation.EMAIL_INVALID + "}")
    @Size(max = 255, message = "{" + MessageKeys.Validation.EMAIL_SIZE + "}")
    private String email;

    @JsonProperty("firstName")
    @NotBlank(message = "{" + MessageKeys.Validation.FIRST_NAME_REQUIRED + "}")
    @Size(min = 1, max = 100, message = "{" + MessageKeys.Validation.FIRST_NAME_SIZE + "}")
    private String firstName;

    @JsonProperty("lastName")
    @NotBlank(message = "{" + MessageKeys.Validation.LAST_NAME_REQUIRED + "}")
    @Size(min = 1, max = 100, message = "{" + MessageKeys.Validation.LAST_NAME_SIZE + "}")
    private String lastName;

    @JsonProperty("birthDate")
    @NotNull(message = "{" + MessageKeys.Validation.BIRTH_DATE_REQUIRED + "}")
    @Past(message = "{" + MessageKeys.Validation.BIRTH_DATE_PAST + "}")
    private LocalDate birthDate;

    // Default constructor for JSON deserialization (required)
    public CreateUserRequest() {}

    // Full constructor for programmatic creation
    public CreateUserRequest(String email, String firstName, String lastName, LocalDate birthDate) {
        this.email = email;
        this.firstName = firstName;
        this.lastName = lastName;
        this.birthDate = birthDate;
    }

    // Convert to domain command object
    public CreateUserCommand toCommand() {
        return CreateUserCommand.builder()
            .email(email)
            .firstName(firstName)
            .lastName(lastName)
            .birthDate(birthDate)
            .build();
    }

    // Standard getters and setters
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }

    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }

    public LocalDate getBirthDate() { return birthDate; }
    public void setBirthDate(LocalDate birthDate) { this.birthDate = birthDate; }

    // Validation helper methods for business logic
    public boolean hasValidNames() {
        return firstName != null && lastName != null && 
               !firstName.trim().isEmpty() && !lastName.trim().isEmpty();
    }

    @Override
    public String toString() {
        return String.format("CreateUserRequest{email='%s', firstName='%s', lastName='%s'}", 
                           email, firstName, lastName);
    }
}
```

**Update Request DTO with Partial Updates**:
```java
package com.copilot.quarkus.infrastructure.web.dto;

import com.copilot.quarkus.infrastructure.i18n.MessageKeys;
import jakarta.validation.constraints.Size;
import jakarta.validation.constraints.Email;
import com.fasterxml.jackson.annotation.JsonProperty;
import io.quarkus.runtime.annotations.RegisterForReflection;
import java.time.LocalDate;
import java.util.Optional;

/**
 * Request DTO for updating an existing User.
 * 
 * <p>Update DTOs support partial updates where only provided fields
 * are updated, maintaining null-safety for optional modifications.
 */
@RegisterForReflection
public class UpdateUserRequest {

    @JsonProperty("firstName")
    @Size(min = 1, max = 100, message = "{" + MessageKeys.Validation.FIRST_NAME_SIZE + "}")
    private String firstName;

    @JsonProperty("lastName")
    @Size(min = 1, max = 100, message = "{" + MessageKeys.Validation.LAST_NAME_SIZE + "}")
    private String lastName;

    @JsonProperty("birthDate")
    @Past(message = "{" + MessageKeys.Validation.BIRTH_DATE_PAST + "}")
    private LocalDate birthDate;

    // Default constructor
    public UpdateUserRequest() {}

    // Constructor with all optional fields
    public UpdateUserRequest(String firstName, String lastName, LocalDate birthDate) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.birthDate = birthDate;
    }

    // Convert to domain update command
    public UpdateUserCommand toCommand(String userId) {
        UpdateUserCommand.Builder builder = UpdateUserCommand.builder().userId(userId);
        
        if (hasFirstName()) builder.firstName(firstName);
        if (hasLastName()) builder.lastName(lastName);
        if (hasBirthDate()) builder.birthDate(birthDate);
        
        return builder.build();
    }

    // Helper methods to check if fields should be updated (null-safe)
    public boolean hasFirstName() { 
        return firstName != null && !firstName.trim().isEmpty(); 
    }
    
    public boolean hasLastName() { 
        return lastName != null && !lastName.trim().isEmpty(); 
    }
    
    public boolean hasBirthDate() { 
        return birthDate != null; 
    }
    
    public boolean hasAnyUpdateField() {
        return hasFirstName() || hasLastName() || hasBirthDate();
    }

    // Getters and setters
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }

    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }

    public LocalDate getBirthDate() { return birthDate; }
    public void setBirthDate(LocalDate birthDate) { this.birthDate = birthDate; }
}
```

## 2. Response DTO Pattern

**Immutable Response DTO using Records**:
```java
package com.copilot.quarkus.infrastructure.web.dto;

import com.copilot.quarkus.domain.user.model.User;
import com.fasterxml.jackson.annotation.JsonProperty;
import io.quarkus.runtime.annotations.RegisterForReflection;
import java.time.LocalDateTime;
import java.time.LocalDate;

/**
 * Response DTO for User data using Java record for immutability.
 *
 * <p>This DTO represents the public API contract for User data
 * and excludes sensitive internal information such as internal IDs,
 * audit fields, and business logic state.
 */
@RegisterForReflection
public record UserResponse(
    @JsonProperty("id") String id,
    @JsonProperty("email") String email,
    @JsonProperty("firstName") String firstName,
    @JsonProperty("lastName") String lastName,
    @JsonProperty("fullName") String fullName,
    @JsonProperty("birthDate") LocalDate birthDate,
    @JsonProperty("age") Integer age,
    @JsonProperty("createdAt") LocalDateTime createdAt,
    @JsonProperty("updatedAt") LocalDateTime updatedAt
) {
    
    /**
     * Factory method to create response from domain entity.
     * 
     * @param user Domain user entity
     * @return UserResponse DTO
     */
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId().toString(),
            user.getEmail(),
            user.getFirstName(),
            user.getLastName(),
            user.getFirstName() + " " + user.getLastName(),
            user.getBirthDate(),
            user.calculateAge(),
            user.getCreatedAt(),
            user.getUpdatedAt()
        );
    }
    
    /**
     * Create minimal response with essential fields only.
     * Useful for list operations or when full details are not needed.
     */
    public static UserResponse minimal(User user) {
        return new UserResponse(
            user.getId().toString(),
            user.getEmail(),
            user.getFirstName(),
            user.getLastName(),
            user.getFirstName() + " " + user.getLastName(),
            null, // birthDate excluded
            null, // age excluded
            user.getCreatedAt(),
            user.getUpdatedAt()
        );
    }
}
```

**List Response DTO Pattern**:
```java
package com.copilot.quarkus.infrastructure.web.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import io.quarkus.runtime.annotations.RegisterForReflection;
import java.util.List;

/**
 * Response DTO for paginated User list data.
 * 
 * <p>Provides pagination metadata along with the actual data
 * to support efficient client-side pagination and navigation.
 */
@RegisterForReflection
public record UserListResponse(
    @JsonProperty("users") List<UserResponse> users,
    @JsonProperty("pagination") PaginationInfo pagination
) {
    
    public static UserListResponse from(List<User> users, int page, int size, long totalElements) {
        List<UserResponse> userResponses = users.stream()
            .map(UserResponse::from)
            .toList();
        
        PaginationInfo pagination = new PaginationInfo(
            page,
            size,
            totalElements,
            (int) Math.ceil((double) totalElements / size)
        );
        
        return new UserListResponse(userResponses, pagination);
    }
    
    @RegisterForReflection
    public record PaginationInfo(
        @JsonProperty("page") int page,
        @JsonProperty("size") int size,
        @JsonProperty("totalElements") long totalElements,
        @JsonProperty("totalPages") int totalPages
    ) {}
}
```

## 3. Message Keys Management

**Centralized Message Key Constants**:
```java
package com.copilot.quarkus.infrastructure.i18n;

/**
 * Centralized message keys for validation and API responses.
 * 
 * <p>All validation messages and API documentation MUST use these keys
 * to ensure consistent internationalization across the application.
 * 
 * <p>Key naming convention: [category].[entity/field].[rule/action]
 */
public final class MessageKeys {

    private MessageKeys() {} // Utility class - prevent instantiation

    public static final class Validation {
        // Email validation keys
        public static final String EMAIL_REQUIRED = "validation.email.required";
        public static final String EMAIL_INVALID = "validation.email.invalid";
        public static final String EMAIL_SIZE = "validation.email.size";
        public static final String EMAIL_UNIQUE = "validation.email.unique";

        // Name validation keys
        public static final String FIRST_NAME_REQUIRED = "validation.firstName.required";
        public static final String FIRST_NAME_SIZE = "validation.firstName.size";
        public static final String LAST_NAME_REQUIRED = "validation.lastName.required";
        public static final String LAST_NAME_SIZE = "validation.lastName.size";

        // Date validation keys
        public static final String BIRTH_DATE_REQUIRED = "validation.birthDate.required";
        public static final String BIRTH_DATE_PAST = "validation.birthDate.past";
        public static final String BIRTH_DATE_VALID_AGE = "validation.birthDate.validAge";

        // Generic validation keys
        public static final String FIELD_REQUIRED = "validation.field.required";
        public static final String FIELD_SIZE = "validation.field.size";
        public static final String FIELD_INVALID = "validation.field.invalid";
        public static final String FIELD_POSITIVE = "validation.field.positive";
    }

    public static final class Api {
        // Standard HTTP response descriptions
        public static final String RESPONSE_200_DESCRIPTION = "api.response.200.description";
        public static final String RESPONSE_201_DESCRIPTION = "api.response.201.description";
        public static final String RESPONSE_204_DESCRIPTION = "api.response.204.description";
        public static final String RESPONSE_400_DESCRIPTION = "api.response.400.description";
        public static final String RESPONSE_404_DESCRIPTION = "api.response.404.description";
        public static final String RESPONSE_500_DESCRIPTION = "api.response.500.description";

        // User API operation summaries
        public static final String USER_CREATE_SUMMARY = "api.user.create.summary";
        public static final String USER_GET_SUMMARY = "api.user.get.summary";
        public static final String USER_GET_ALL_SUMMARY = "api.user.getAll.summary";
        public static final String USER_UPDATE_SUMMARY = "api.user.update.summary";
        public static final String USER_DELETE_SUMMARY = "api.user.delete.summary";
        
        // User API descriptions
        public static final String USER_CREATE_DESCRIPTION = "api.user.create.description";
        public static final String USER_UPDATE_DESCRIPTION = "api.user.update.description";
    }

    public static final class Log {
        // User operation log messages
        public static final String USER_CREATING = "log.user.creating";
        public static final String USER_CREATED = "log.user.created";
        public static final String USER_RETRIEVING_ID = "log.user.retrieving.id";
        public static final String USER_RETRIEVING_ALL = "log.user.retrieving.all";
        public static final String USER_UPDATED = "log.user.updated";
        public static final String USER_DELETED = "log.user.deleted";
        
        // Error log messages
        public static final String USER_NOT_FOUND = "log.user.notFound";
        public static final String USER_VALIDATION_FAILED = "log.user.validationFailed";
        public static final String USER_CREATION_FAILED = "log.user.creationFailed";
    }
}
```

## 4. Custom Validation Implementation

**Business Rule Validator Example**:
```java
package com.copilot.quarkus.infrastructure.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.LocalDate;
import java.time.Period;

/**
 * Custom validator for business email rules.
 * 
 * <p>Validates that email addresses meet business requirements
 * beyond basic format validation (e.g., domain restrictions,
 * business rules for acceptable email providers).
 */
@ApplicationScoped
public class BusinessEmailValidator implements ConstraintValidator<BusinessEmail, String> {
    
    private static final Set<String> ALLOWED_DOMAINS = Set.of(
        "company.com", "example.com", "partner.org"
    );
    
    private static final Set<String> BLOCKED_DOMAINS = Set.of(
        "tempmail.org", "10minutemail.com", "guerrillamail.com"
    );
    
    @Override
    public void initialize(BusinessEmail constraintAnnotation) {
        // Initialization if needed
    }
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null || email.isEmpty()) {
            return true; // Let @NotBlank handle empty validation
        }
        
        String domain = extractDomain(email);
        if (domain == null) {
            return false; // Invalid email format
        }
        
        // Check if domain is explicitly blocked
        if (BLOCKED_DOMAINS.contains(domain.toLowerCase())) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(
                "{validation.email.domain.blocked}"
            ).addConstraintViolation();
            return false;
        }
        
        // For strict environments, check allowed domains
        // if (!ALLOWED_DOMAINS.contains(domain.toLowerCase())) {
        //     return false;
        // }
        
        return true;
    }
    
    private String extractDomain(String email) {
        int atIndex = email.lastIndexOf('@');
        if (atIndex > 0 && atIndex < email.length() - 1) {
            return email.substring(atIndex + 1);
        }
        return null;
    }
}

/**
 * Custom constraint annotation for business email validation.
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = BusinessEmailValidator.class)
public @interface BusinessEmail {
    String message() default "{validation.email.business.invalid}";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

## 5. Message Bundle Configuration

**Complete Message Properties** (`messages_en.properties`):
```properties
# Validation Messages - English
validation.email.required=Email address is required
validation.email.invalid=Email address format is invalid
validation.email.size=Email address must not exceed 255 characters
validation.email.unique=Email address is already registered
validation.email.business.invalid=Email address does not meet business requirements
validation.email.domain.blocked=Email domain is not allowed

validation.firstName.required=First name is required
validation.firstName.size=First name must be between 1 and 100 characters
validation.lastName.required=Last name is required
validation.lastName.size=Last name must be between 1 and 100 characters

validation.birthDate.required=Birth date is required
validation.birthDate.past=Birth date must be in the past
validation.birthDate.validAge=Age must be between 13 and 150 years

validation.field.required=This field is required
validation.field.size=Field length is invalid
validation.field.invalid=Field value is invalid
validation.field.positive=Field value must be positive

# API Documentation Messages
api.response.200.description=Operation completed successfully
api.response.201.description=Resource created successfully
api.response.204.description=Operation completed successfully with no content
api.response.400.description=Invalid request data provided
api.response.404.description=Requested resource was not found
api.response.500.description=Internal server error occurred

api.user.create.summary=Create a new user account
api.user.create.description=Creates a new user account with the provided information
api.user.get.summary=Retrieve user by ID
api.user.getAll.summary=Retrieve all users with pagination
api.user.update.summary=Update existing user information
api.user.update.description=Updates an existing user's information with partial data
api.user.delete.summary=Delete user account

# Logging Messages
log.user.creating=Creating new user with email: {0}
log.user.created=Successfully created user with ID: {0}
log.user.retrieving.id=Retrieving user with ID: {0}
log.user.retrieving.all=Retrieving all users with pagination
log.user.updated=Successfully updated user with ID: {0}
log.user.deleted=Successfully deleted user with ID: {0}

log.user.notFound=User not found with ID: {0}
log.user.validationFailed=User data validation failed: {0}
log.user.creationFailed=Failed to create user: {0}
```

**Validation Commands**:
```bash
# Test DTO validation with comprehensive scenarios
./mvnw test -Dtest=CreateUserRequestValidationTest

# Test response DTO serialization
./mvnw test -Dtest=UserResponseSerializationTest

# Test message key resolution
./mvnw test -Dtest=MessageKeyValidationTest

# Test custom validators
./mvnw test -Dtest=BusinessEmailValidatorTest

# Test partial update scenarios
./mvnw test -Dtest=UpdateUserRequestTest

# Verify GraalVM native compilation with DTOs
./mvnw package -Pnative -DskipTests
```

# Additional Recommendations:

- **RECOMMENDED**: Use MapStruct for compile-time DTO mapping with type safety — see MapStruct reference guide (https://mapstruct.org/documentation/stable/reference/html/).
- **RECOMMENDED**: Implement validation groups (`jakarta.validation.groups`) to apply context-specific constraints — see Hibernate Validator grouping documentation (https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#chapter-groups).
- **RECOMMENDED**: Use Java records for all response DTOs to leverage immutability and pattern matching — see JEP 395 (https://openjdk.org/jeps/395).
- **RECOMMENDED**: Ensure DTOs generate comprehensive OpenAPI documentation via SmallRye OpenAPI — see Quarkus OpenAPI guide (https://quarkus.io/guides/openapi-swaggerui).
- **RECOMMENDED**: Implement class-level `@ScriptAssert` or custom validators for cross-field validation rules — see Hibernate Validator custom constraints (https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#validator-customconstraints).

# Success Criteria:

**Completion Gate**: The following criteria **MUST** all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Request DTO Classes | `grep -r "class Create.*Request" */src/main/java` | Request DTOs with validation annotations |
| Response DTO Records | `grep -r "public record.*Response" */src/main/java` | Java record response DTOs |
| Bean Validation Annotations | `grep -r "@NotBlank" */src/main/java` | Validation annotations on DTO fields |
| Internationalized Messages | `grep -r "MessageKeys" */src/main/java` | Message keys, not hardcoded strings |
| Message Properties | `find . -name "messages*.properties"` | Resource bundle files present |
| RegisterForReflection | `grep -r "@RegisterForReflection" */src/main/java` | GraalVM compatibility |
| Validation Tests | `./mvnw test -Dtest=*Validation*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Data Transfer & Validation..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for Bean Validation dependency
if find . -name "pom.xml" -exec grep -q "jakarta.validation\|hibernate-validator\|quarkus-hibernate-validator" {} + 2>/dev/null; then
  echo "✅ Bean Validation dependency present in pom.xml"
else
  echo "❌ Bean Validation dependency not found in pom.xml"
  FAIL=1
fi

# Check for Request DTO classes
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "class Create.*Request\|class Update.*Request" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Request DTO classes found"
else
  echo "❌ No Request DTO classes (Create*Request/Update*Request) found"
  FAIL=1
fi

# Check for Response DTO records
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "public record.*Response" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Java record Response DTOs found"
else
  echo "❌ No Java record Response DTOs found"
  FAIL=1
fi

# Check for validation annotations
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "@NotNull\|@NotBlank\|@Size\|@Pattern\|@Valid\|@Email" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Validation annotations in use"
else
  echo "❌ No Bean Validation annotations found"
  FAIL=1
fi

# Check for internationalized message keys (not hardcoded)
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "MessageKeys\|message.*validation\." "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Internationalized validation message keys found"
else
  echo "❌ No internationalized message keys found"
  FAIL=1
fi

# Check for message properties files
if find . -name "messages*.properties" 2>/dev/null | grep -q .; then
  echo "✅ Message resource bundle files found"
else
  echo "❌ No messages*.properties resource bundle files found"
  FAIL=1
fi

# Check for RegisterForReflection on DTOs
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "@RegisterForReflection" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ @RegisterForReflection annotations found (GraalVM support)"
else
  echo "❌ No @RegisterForReflection found for native compilation"
  FAIL=1
fi

# Run validation tests
if ./mvnw test -Dtest="*Validation*,*Dto*,*DTO*" -q 2>/dev/null; then
  echo "✅ Validation tests pass"
else
  echo "❌ Validation tests failed or not found"
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
  echo "❌ Data Transfer & Validation validation failed"
  exit 1
fi
echo "✅ All Data Transfer & Validation criteria met"
```
