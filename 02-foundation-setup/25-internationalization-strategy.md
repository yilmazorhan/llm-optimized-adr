# Issue: Internationalization Strategy for Multi-Language Support

Reactive microservices require comprehensive i18n/l10n support with locale context propagation across service boundaries while maintaining performance and architectural boundaries.

# Decision:

Implement externalized message management with locale-independent error codes and reactive context propagation. All user-facing messages and system output messages externalized to resource bundles, business logic remains locale-agnostic.

**Required Components**:
- **Locale-Independent Error Codes**: Structured codes following `[DOMAIN]-[CATEGORY]-[SEQUENCE]` pattern
- **Resource Bundle Management**: Hierarchical message organization by domain and locale
- **Reactive Context Propagation**: Locale context maintained across async boundaries
- **Thread-Safe Message Resolution**: Cached, concurrent message resolution service
- **Hexagonal Architecture Integration**: Localization restricted to infrastructure adapters

# Constraints:

- All i18n components **MUST** be compatible with GraalVM native compilation, including resource bundle loading and reflection configuration.
- Message bundle caching **MUST NOT** consume unbounded memory; cache size **MUST** be configurable or bounded.
- Bundle loading and cache initialization **MUST NOT** block application startup beyond measurable thresholds.
- Locale context propagation **MUST NOT** block the event loop or introduce measurable latency on the I/O thread.
- All message resolution components **MUST** be thread-safe for concurrent access without synchronized blocks.
- Locale context **MUST** propagate correctly across async and reactive boundaries using Quarkus Context Propagation.
- Business logic in the domain layer **MUST NOT** reference locale objects, resource bundles, or translated strings.
- Hexagonal architecture ports **MUST NOT** expose locale-specific interfaces.
- Only infrastructure adapters **MAY** perform locale detection and message localization.

# Alternatives:

### 1. Database-Driven i18n

Storing messages in database tables with locale columns adds a database round-trip for every message resolution call. The Java `ResourceBundle` specification (https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ResourceBundle.html) caches bundles in memory after the first load, producing sub-microsecond lookups. Database-backed resolution replaces this with network I/O per request. Quarkus RESTEasy Reactive serializes responses on the I/O thread (https://quarkus.io/guides/resteasy-reactive#serialization), so blocking database calls during message resolution would stall the event loop.

### 2. External Translation Service (e.g., Google Cloud Translation API)

Integrating a runtime translation API introduces an external HTTP dependency for every error message. The Cloud Translation API pricing page (https://cloud.google.com/translate/pricing) documents per-character billing. Network latency adds 50-200 ms per call compared to in-memory bundle lookups measured in nanoseconds. Service outages would prevent the application from rendering any user-facing error messages, violating availability requirements.

# Rationale:

Externalized resource bundles decouple message content from compiled code. The Java `ResourceBundle` specification (https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ResourceBundle.html) defines hierarchical locale fallback (e.g., `messages_es_MX` -> `messages_es` -> `messages`), enabling regional customization without duplicating all keys.

Domain exceptions carry structured error codes (e.g., `USER-BUSINESS-002`) without translated text, preserving domain layer purity as required by hexagonal architecture (ADR-10). The Quarkus SmallRye Context Propagation extension (https://quarkus.io/guides/context-propagation) captures and reattaches thread-local state across reactive operator boundaries, ensuring consistent locale during message resolution in Mutiny pipelines.

The `@RegisterForReflection` annotation on message-related DTOs ensures GraalVM native image compilation includes the classes required for JSON serialization (https://quarkus.io/guides/writing-native-applications-tips#registering-for-reflection). The `.properties` file format is supported by professional translation management tools (Crowdin, Transifex, Lokalise), enabling external translation workflows without code changes.

# Implementation Guidelines:

## Naming Conventions:
- Error codes: `[DOMAIN]-[CATEGORY]-[SEQUENCE]` (e.g., `USER-VALIDATION-001`)
- Resource bundles: `[domain]-messages_[locale].properties`
- Message keys: `[error-code].[type]` (e.g., `USER-BUSINESS-002.title`)
- Context providers: `[Domain]LocaleContextProvider`
- Message resolvers: `[Domain]MessageResolver`
- Locale detectors: `[Protocol]LocaleDetectionService`

## 1. Domain Layer: Locale-Independent Error Codes

**Structured Error Code Implementation**:
```java
package com.copilot.quarkus.domain.user.exception;

import com.copilot.quarkus.shared.domain.exception.BusinessRuleViolationException;
import java.util.Map;

/**
 * Domain exception for email uniqueness violation.
 * 
 * <p>Carries only technical context and error code, never user-facing messages.
 * Message localization handled by infrastructure adapters.
 */
public class EmailAlreadyExistsException extends BusinessRuleViolationException {
    
    public static final String ERROR_CODE = "USER-BUSINESS-002";
    
    public EmailAlreadyExistsException(String email) {
        super(ERROR_CODE, Map.of("email", email));
    }
    
    public EmailAlreadyExistsException(String email, Throwable cause) {
        super(ERROR_CODE, Map.of("email", email), cause);
    }
    
    public String getEmail() {
        return (String) getErrorContext().get("email");
    }
}
```

**Error Code Registry**:
```java
package com.copilot.quarkus.shared.domain.exception;

/**
 * Centralized registry of all domain error codes.
 * 
 * <p>All error codes MUST follow the pattern: [DOMAIN]-[CATEGORY]-[SEQUENCE]
 * This ensures consistent error identification across all services and locales.
 */
public final class ErrorCodes {
    
    private ErrorCodes() {} // Utility class
    
    // User Domain Error Codes
    public static final class User {
        public static final String VALIDATION_EMAIL_FORMAT = "USER-VALIDATION-001";
        public static final String BUSINESS_EMAIL_EXISTS = "USER-BUSINESS-002";
        public static final String BUSINESS_INVALID_AGE = "USER-BUSINESS-003";
        public static final String WORKFLOW_INVALID_STATE = "USER-WORKFLOW-004";
        public static final String INFRASTRUCTURE_DATABASE_ERROR = "USER-INFRASTRUCTURE-005";
    }
    
    // Order Domain Error Codes  
    public static final class Order {
        public static final String VALIDATION_QUANTITY_INVALID = "ORDER-VALIDATION-001";
        public static final String BUSINESS_INSUFFICIENT_INVENTORY = "ORDER-BUSINESS-002";
        public static final String WORKFLOW_INVALID_TRANSITION = "ORDER-WORKFLOW-003";
        public static final String INFRASTRUCTURE_PAYMENT_FAILED = "ORDER-INFRASTRUCTURE-004";
    }
    
    // Add other domains as needed
}
```

## 2. Resource Bundle Management

**Hierarchical Message Organization**:
```
src/main/resources/i18n/
├── messages_en.properties              # Global messages
├── messages_es.properties
├── messages_fr.properties
├── messages_de.properties
└── domain/
    ├── user-messages_en.properties    # User domain specific
    ├── user-messages_es.properties
    ├── user-messages_fr.properties
    ├── order-messages_en.properties   # Order domain specific
    └── order-messages_es.properties
```

**Complete Message Bundle Template** (`user-messages_en.properties`):
```properties
# User Domain Messages - English
# Error Code: USER-BUSINESS-002 (Email Already Exists)
USER-BUSINESS-002.title=Email Already Exists
USER-BUSINESS-002.detail=The email address ''{0}'' is already registered in the system
USER-BUSINESS-002.action=Please use a different email address or sign in with the existing account

# Error Code: USER-VALIDATION-001 (Email Format Invalid)  
USER-VALIDATION-001.title=Invalid Email Format
USER-VALIDATION-001.detail=The email address ''{0}'' does not have a valid format
USER-VALIDATION-001.action=Please enter a valid email address (example: user@domain.com)

# Error Code: USER-BUSINESS-003 (Invalid Age)
USER-BUSINESS-003.title=Invalid Age Restriction
USER-BUSINESS-003.detail=Users must be between {0} and {1} years old to register
USER-BUSINESS-003.action=Please verify your birth date and ensure you meet the age requirements

# Success Messages
USER-OPERATION-SUCCESS.title=Operation Successful
USER-OPERATION-SUCCESS.detail=User operation completed successfully
USER-OPERATION-SUCCESS.action=You can proceed with the next step

# API Documentation Messages
USER-CREATE-API.summary=Create a new user account
USER-CREATE-API.description=Creates a new user account with the provided information
USER-UPDATE-API.summary=Update existing user information
USER-UPDATE-API.description=Updates an existing user''s information with partial data
```

**Spanish Translation Example** (`user-messages_es.properties`):
```properties
# Mensajes del Dominio Usuario - Español
# Código de Error: USER-BUSINESS-002 (Correo Electrónico Ya Existe)
USER-BUSINESS-002.title=Correo Electrónico Ya Existe
USER-BUSINESS-002.detail=La dirección de correo electrónico ''{0}'' ya está registrada en el sistema
USER-BUSINESS-002.action=Por favor, use una dirección de correo diferente o inicie sesión con la cuenta existente

# Código de Error: USER-VALIDATION-001 (Formato de Correo Inválido)
USER-VALIDATION-001.title=Formato de Correo Electrónico Inválido
USER-VALIDATION-001.detail=La dirección de correo electrónico ''{0}'' no tiene un formato válido
USER-VALIDATION-001.action=Por favor, ingrese una dirección de correo válida (ejemplo: usuario@dominio.com)

# Código de Error: USER-BUSINESS-003 (Edad Inválida)
USER-BUSINESS-003.title=Restricción de Edad Inválida
USER-BUSINESS-003.detail=Los usuarios deben tener entre {0} y {1} años para registrarse
USER-BUSINESS-003.action=Por favor, verifique su fecha de nacimiento y asegúrese de cumplir con los requisitos de edad
```

## 3. Reactive Context Propagation

**Locale Context Provider with Reactive Support**:
```java
package com.copilot.quarkus.infrastructure.i18n;

import io.smallrye.common.annotation.Identifier;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import org.eclipse.microprofile.context.ManagedExecutor;
import org.eclipse.microprofile.context.ThreadContext;

import java.util.Locale;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.function.Supplier;

/**
 * Reactive-safe locale context provider.
 * 
 * <p>Manages locale context across async boundaries using Quarkus Context Propagation.
 * Thread-safe and optimized for high-concurrency reactive operations.
 */
@ApplicationScoped
public class LocaleContextProvider {
    
    private static final String LOCALE_CONTEXT_KEY = "user.locale";
    private final ThreadLocal<Locale> localeHolder = new ThreadLocal<>();
    
    @Inject
    @Identifier("virtualthread-executor")
    ManagedExecutor managedExecutor;
    
    @Inject
    ThreadContext threadContext;
    
    /**
     * Execute operation with specific locale context in reactive stream.
     * 
     * @param locale The locale to set for the operation
     * @param operation The reactive operation to execute
     * @return Uni with locale context maintained
     */
    public <T> Uni<T> withLocale(Locale locale, Supplier<Uni<T>> operation) {
        return Uni.createFrom().deferred(() -> {
            setCurrentLocale(locale);
            return operation.get()
                .eventually(() -> {
                    clearCurrentLocale();
                    return Uni.createFrom().voidItem();
                });
        }).runSubscriptionOn(managedExecutor);
    }
    
    /**
     * Get current locale from context with fallback to default.
     * 
     * @return Current locale or default English locale
     */
    public Locale getCurrentLocale() {
        return Optional.ofNullable(localeHolder.get())
                .orElse(Locale.ENGLISH);
    }
    
    /**
     * Set locale in current thread context.
     * 
     * @param locale Locale to set
     */
    public void setCurrentLocale(Locale locale) {
        localeHolder.set(locale != null ? locale : Locale.ENGLISH);
    }
    
    /**
     * Clear locale from current thread context.
     */
    public void clearCurrentLocale() {
        localeHolder.remove();
    }
    
    /**
     * Create contextualized supplier that captures current locale.
     * 
     * @param operation Operation to contextualize
     * @return Contextualized supplier
     */
    public <T> Supplier<T> withCurrentLocale(Supplier<T> operation) {
        Locale currentLocale = getCurrentLocale();
        return () -> {
            Locale previousLocale = getCurrentLocale();
            try {
                setCurrentLocale(currentLocale);
                return operation.get();
            } finally {
                setCurrentLocale(previousLocale);
            }
        };
    }
}
```

## 4. Thread-Safe Message Resolution

**High-Performance Message Resolver**:
```java
package com.copilot.quarkus.infrastructure.i18n;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import io.quarkus.runtime.annotations.RegisterForReflection;

import java.text.MessageFormat;
import java.util.Locale;
import java.util.MissingResourceException;
import java.util.ResourceBundle;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Thread-safe, cached message resolution service.
 * 
 * <p>Provides high-performance message localization with fallback support.
 * Optimized for concurrent access in reactive environments.
 */
@ApplicationScoped
@RegisterForReflection
public class MessageResolver {
    
    private final ConcurrentHashMap<String, ResourceBundle> bundleCache = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, MessageFormat> formatCache = new ConcurrentHashMap<>();
    
    @Inject
    LocaleContextProvider localeProvider;
    
    /**
     * Resolve localized message with parameters.
     * 
     * @param errorCode Error code to resolve
     * @param parameters Message parameters
     * @return Localized message object
     */
    public LocalizedMessage resolve(String errorCode, Object... parameters) {
        Locale locale = localeProvider.getCurrentLocale();
        return resolve(errorCode, locale, parameters);
    }
    
    /**
     * Resolve localized message for specific locale.
     * 
     * @param errorCode Error code to resolve
     * @param locale Target locale
     * @param parameters Message parameters  
     * @return Localized message object
     */
    public LocalizedMessage resolve(String errorCode, Locale locale, Object... parameters) {
        try {
            ResourceBundle bundle = getResourceBundle(locale);
            
            String title = resolveMessage(bundle, errorCode + ".title", locale, parameters);
            String detail = resolveMessage(bundle, errorCode + ".detail", locale, parameters);
            String action = resolveMessage(bundle, errorCode + ".action", locale, parameters);
            
            return LocalizedMessage.builder()
                .errorCode(errorCode)
                .locale(locale)
                .title(title)
                .detail(detail)
                .action(action)
                .build();
                
        } catch (MissingResourceException e) {
            // Fallback to English if resource not found
            if (!Locale.ENGLISH.equals(locale)) {
                return resolve(errorCode, Locale.ENGLISH, parameters);
            }
            
            // Ultimate fallback to generic message
            return createFallbackMessage(errorCode, locale);
        }
    }
    
    private ResourceBundle getResourceBundle(Locale locale) {
        String key = locale.toLanguageTag();
        return bundleCache.computeIfAbsent(key, k -> {
            try {
                return ResourceBundle.getBundle("i18n.messages", locale, 
                    Thread.currentThread().getContextClassLoader());
            } catch (MissingResourceException e) {
                // Try domain-specific bundles
                return loadDomainBundle(locale);
            }
        });
    }
    
    private ResourceBundle loadDomainBundle(Locale locale) {
        // Try user domain bundle first
        try {
            return ResourceBundle.getBundle("i18n.domain.user-messages", locale,
                Thread.currentThread().getContextClassLoader());
        } catch (MissingResourceException e) {
            // Fallback to base bundle
            return ResourceBundle.getBundle("i18n.messages", Locale.ENGLISH,
                Thread.currentThread().getContextClassLoader());
        }
    }
    
    private String resolveMessage(ResourceBundle bundle, String key, Locale locale, Object... parameters) {
        try {
            String pattern = bundle.getString(key);
            if (parameters.length == 0) {
                return pattern;
            }
            
            // Use cached MessageFormat for performance
            String formatKey = locale.toLanguageTag() + ":" + key;
            MessageFormat format = formatCache.computeIfAbsent(formatKey, 
                k -> new MessageFormat(pattern, locale));
            
            return format.format(parameters);
            
        } catch (MissingResourceException e) {
            return String.format("[%s]", key); // Indicate missing translation
        }
    }
    
    private LocalizedMessage createFallbackMessage(String errorCode, Locale locale) {
        return LocalizedMessage.builder()
            .errorCode(errorCode)
            .locale(locale)
            .title("System Error")
            .detail(String.format("An error occurred with code: %s", errorCode))
            .action("Please contact system administrator")
            .build();
    }
}
```

**Localized Message Data Structure**:
```java
package com.copilot.quarkus.infrastructure.i18n;

import com.fasterxml.jackson.annotation.JsonProperty;
import io.quarkus.runtime.annotations.RegisterForReflection;
import java.util.Locale;

/**
 * Immutable localized message representation.
 * 
 * <p>Contains all localized content for an error or information message
 * with structured format suitable for API responses.
 */
@RegisterForReflection
public record LocalizedMessage(
    @JsonProperty("errorCode") String errorCode,
    @JsonProperty("locale") String locale,
    @JsonProperty("title") String title,
    @JsonProperty("detail") String detail,
    @JsonProperty("action") String action
) {
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private String errorCode;
        private Locale locale;
        private String title;
        private String detail;
        private String action;
        
        public Builder errorCode(String errorCode) {
            this.errorCode = errorCode;
            return this;
        }
        
        public Builder locale(Locale locale) {
            this.locale = locale;
            return this;
        }
        
        public Builder title(String title) {
            this.title = title;
            return this;
        }
        
        public Builder detail(String detail) {
            this.detail = detail;
            return this;
        }
        
        public Builder action(String action) {
            this.action = action;
            return this;
        }
        
        public LocalizedMessage build() {
            return new LocalizedMessage(
                errorCode,
                locale != null ? locale.toLanguageTag() : Locale.ENGLISH.toLanguageTag(),
                title,
                detail,
                action
            );
        }
    }
}
```

## 5. REST Layer Integration

**HTTP Locale Detection Service**:
```java
package com.copilot.quarkus.infrastructure.web.i18n;

import io.vertx.ext.web.RoutingContext;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Arrays;
import java.util.List;
import java.util.Locale;
import java.util.Optional;

/**
 * HTTP-specific locale detection with multiple strategies.
 * 
 * <p>Detects user locale from various HTTP sources with priority-based fallback.
 */
@ApplicationScoped
public class HttpLocaleDetectionService {
    
    private static final List<Locale> SUPPORTED_LOCALES = Arrays.asList(
        Locale.ENGLISH,
        Locale.forLanguageTag("es"),
        Locale.forLanguageTag("fr"),
        Locale.forLanguageTag("de")
    );
    
    /**
     * Detect locale from HTTP request with fallback strategy.
     * 
     * Priority: 1. Query parameter 2. Accept-Language header 3. Default
     * 
     * @param routingContext HTTP routing context
     * @return Detected locale
     */
    public Locale detectLocale(RoutingContext routingContext) {
        // 1. Check query parameter 'lang'
        return Optional.ofNullable(routingContext.request().getParam("lang"))
            .map(Locale::forLanguageTag)
            .filter(this::isSupportedLocale)
            .or(() -> 
                // 2. Parse Accept-Language header
                Optional.ofNullable(routingContext.request().getHeader("Accept-Language"))
                    .map(this::parseAcceptLanguage)
                    .filter(this::isSupportedLocale)
            )
            .orElse(Locale.ENGLISH); // 3. Default fallback
    }
    
    /**
     * Parse Accept-Language header to extract preferred locale.
     * 
     * @param acceptLanguage Accept-Language header value
     * @return Preferred locale or null if parsing fails
     */
    private Locale parseAcceptLanguage(String acceptLanguage) {
        if (acceptLanguage == null || acceptLanguage.trim().isEmpty()) {
            return null;
        }
        
        // Simple implementation - takes first language tag
        // Production implementation should parse quality values
        String[] languages = acceptLanguage.split(",");
        if (languages.length > 0) {
            String primaryLanguage = languages[0].trim().split(";")[0]; // Remove quality value
            return Locale.forLanguageTag(primaryLanguage);
        }
        
        return null;
    }
    
    /**
     * Check if locale is supported by the application.
     * 
     * @param locale Locale to check
     * @return true if supported, false otherwise
     */
    private boolean isSupportedLocale(Locale locale) {
        if (locale == null) return false;
        
        return SUPPORTED_LOCALES.stream()
            .anyMatch(supported -> 
                supported.getLanguage().equals(locale.getLanguage()));
    }
}
```

**Localized Exception Mapper**:
```java
package com.copilot.quarkus.infrastructure.web.exception;

import com.copilot.quarkus.infrastructure.i18n.LocaleContextProvider;
import com.copilot.quarkus.infrastructure.i18n.MessageResolver;
import com.copilot.quarkus.infrastructure.i18n.LocalizedMessage;
import com.copilot.quarkus.shared.domain.exception.DomainException;
import jakarta.inject.Inject;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;
import java.net.URI;
import java.util.Locale;

/**
 * Exception mapper providing localized error responses.
 * 
 * <p>Transforms domain exceptions into RFC 9457 Problem Details
 * with localized content based on current user locale.
 */
@Provider
public class LocalizedExceptionMapper implements ExceptionMapper<DomainException> {
    
    @Inject
    MessageResolver messageResolver;
    
    @Inject
    LocaleContextProvider localeProvider;
    
    @Override
    public Response toResponse(DomainException exception) {
        Locale locale = localeProvider.getCurrentLocale();
        LocalizedMessage message = messageResolver.resolve(
            exception.getErrorCode(), 
            locale, 
            exception.getParameters().values().toArray()
        );
        
        ProblemDetail problem = ProblemDetail.forStatus(
            determineHttpStatus(exception.getErrorCode())
        );
        
        problem.setType(URI.create(
            String.format("https://api.copilot-quarkus.com/problems/%s", exception.getErrorCode())
        ));
        problem.setTitle(message.title());
        problem.setDetail(message.detail());
        problem.setProperty("errorCode", exception.getErrorCode());
        problem.setProperty("action", message.action());
        problem.setProperty("locale", message.locale());
        problem.setProperty("correlationId", generateCorrelationId());
        
        return Response.status(problem.getStatus())
            .entity(problem)
            .type("application/problem+json")
            .header("Content-Language", locale.toLanguageTag())
            .build();
    }
    
    private int determineHttpStatus(String errorCode) {
        // Map error codes to HTTP status codes
        return switch (errorCode) {
            case "USER-VALIDATION-001" -> 400; // Bad Request
            case "USER-BUSINESS-002" -> 409;   // Conflict
            case "USER-BUSINESS-003" -> 422;   // Unprocessable Entity
            case "USER-WORKFLOW-004" -> 400;   // Bad Request
            case "USER-INFRASTRUCTURE-005" -> 503; // Service Unavailable
            default -> 500; // Internal Server Error
        };
    }
    
    private String generateCorrelationId() {
        return java.util.UUID.randomUUID().toString().replace("-", "").substring(0, 12);
    }
}
```

## 6. Configuration and Dependencies

**Quarkus Configuration** (`application.properties`):
```properties
# Internationalization Configuration
quarkus.locales=en,es,fr,de
quarkus.default-locale=en

# Resource bundle configuration for GraalVM native compilation
quarkus.native.resources.includes=i18n/**/*.properties,i18n/domain/**/*.properties

# Context propagation for reactive locale handling
quarkus.reactive-messaging.context-propagation=true
quarkus.context-propagation.enabled=true

# Virtual thread executor configuration
quarkus.virtual-threads.enabled=true

# HTTP locale detection configuration
quarkus.http.header."Accept-Language".enabled=true
```

**Maven Dependencies** (`pom.xml`):
```xml
<dependencies>
    <!-- Core Quarkus dependencies -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-resteasy-reactive-jackson</artifactId>
    </dependency>
    
    <!-- Context propagation for reactive locale handling -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-smallrye-context-propagation</artifactId>
    </dependency>
    
    <!-- Virtual threads support (Java 21 feature) -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-virtual-threads</artifactId>
    </dependency>
    
    <!-- Problem Details for HTTP APIs (RFC 9457) -->
    <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-problem-details</artifactId>
    </dependency>
</dependencies>
```

**Validation Commands**:
```bash
# Test message resolution with different locales
./mvnw test -Dtest=MessageResolverTest

# Test reactive context propagation
./mvnw test -Dtest=LocaleContextProviderTest

# Test HTTP locale detection
./mvnw test -Dtest=HttpLocaleDetectionServiceTest

# Test localized exception mapping
./mvnw test -Dtest=LocalizedExceptionMapperTest

# Verify GraalVM native compilation with i18n resources
./mvnw package -Pnative -DskipTests

# Test REST endpoints with different Accept-Language headers
curl -H "Accept-Language: es-ES,es;q=0.9,en;q=0.8" "http://localhost:8080/api/v1/users/invalid"
curl -H "Accept-Language: fr-FR,fr;q=0.9,en;q=0.5" "http://localhost:8080/api/v1/users/invalid"

# Test query parameter locale selection
curl "http://localhost:8080/api/v1/users/invalid?lang=de"
```

# Additional Recommendations:

- **RECOMMENDED**: Use `ResourceBundle.Control` with `TimeToLive` to configure cache expiration -- see Java ResourceBundle.Control documentation (https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ResourceBundle.Control.html).
- **RECOMMENDED**: Integrate translation management via Crowdin CLI for automated sync of `.properties` files -- see Crowdin CLI documentation (https://support.crowdin.com/cli-tool/).
- **RECOMMENDED**: Validate translation completeness at build time with a Maven Enforcer custom rule -- see Maven Enforcer Plugin documentation (https://maven.apache.org/enforcer/maven-enforcer-plugin/).
- **RECOMMENDED**: Use the Quarkus `@MessageBundle` type-safe message API for compile-time key validation -- see Quarkus Qute i18n guide (https://quarkus.io/guides/qute-reference#type-safe-message-bundles).
- **RECOMMENDED**: Log unresolved message keys at WARN level to detect missing translations in production -- see Quarkus logging guide (https://quarkus.io/guides/logging).

# Success Criteria:

**Completion Gate**: The following criteria **MUST** all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Message Bundles | `find . -path "*/resources/i18n/*messages*.properties"` | Default bundle and domain bundles exist |
| Multiple Locales | `find . -name "messages_*.properties" -path "*/resources/*"` | At least 2 locale bundles present |
| Locale Detection | `grep -r "Accept-Language" */src/main/java` | HTTP locale detection implemented |
| Message Resolver | `grep -r "MessageResolver\|@MessageBundle\|ResourceBundle" */src/main/java` | Injected message resolution service |
| Parameterized Messages | `grep -r "{0}\|{1}" */src/main/resources/i18n/` | Parameterized message placeholders |
| Error Code Registry | `grep -r "ErrorCodes\|ERROR_CODE" */src/main/java` | Centralized error code constants |
| RegisterForReflection | `grep -r "@RegisterForReflection" */src/main/java` | GraalVM compatibility on i18n DTOs |
| i18n Tests | `./mvnw test -Dtest=*i18n*,*Locale*,*Message*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Internationalization Strategy..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for message bundles
BUNDLES=$(find . -name "messages*.properties" -path "*/resources/*" 2>/dev/null | wc -l)
if [ "$BUNDLES" -gt 0 ]; then
  echo "✅ Message bundles found: $BUNDLES"
else
  echo "❌ No message bundles found"
  FAIL=1
fi

# Check for multiple locales
LOCALE_COUNT=$(find . -name "messages_*.properties" -path "*/resources/*" 2>/dev/null | sed 's/.*messages_\(.*\)\.properties/\1/' | sort -u | wc -l)
if [ "$LOCALE_COUNT" -ge 2 ]; then
  echo "✅ Multiple locales supported: $LOCALE_COUNT"
else
  echo "❌ Fewer than 2 locale bundles found"
  FAIL=1
fi

# Check for message resolution mechanism
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "MessageResolver\|@MessageBundle\|ResourceBundle" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Message resolution mechanism found"
else
  echo "❌ No message resolution mechanism found"
  FAIL=1
fi

# Check for locale detection from HTTP headers
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "Accept-Language\|LocaleDetection\|detectLocale" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ HTTP locale detection found"
else
  echo "❌ No HTTP locale detection found"
  FAIL=1
fi

# Check for error code registry
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "ErrorCodes\|ERROR_CODE" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Error code registry found"
else
  echo "❌ No error code registry found"
  FAIL=1
fi

# Check for parameterized messages
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "0\|1" "$rp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Parameterized messages found"
else
  echo "❌ No parameterized messages found"
  FAIL=1
fi

# Check for RegisterForReflection on i18n classes
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

# Run i18n tests
if ./mvnw test -Dtest="*i18n*,*Locale*,*Message*" -q 2>/dev/null; then
  echo "✅ i18n tests pass"
else
  echo "❌ i18n tests failed or not found"
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
  echo "❌ Internationalization Strategy validation failed"
  exit 1
fi
echo "✅ All Internationalization Strategy criteria met"
```
