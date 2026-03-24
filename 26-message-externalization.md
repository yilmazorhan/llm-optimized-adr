# Issue: String Message Externalization for Internationalization

Hardcoded strings throughout the codebase in validation messages, logging statements, API documentation, and other components create inconsistencies and translation maintenance challenges. The application requires comprehensive internationalization (i18n) support to serve users in multiple languages.

# Decision:

Implement complete string externalization for all hardcoded text content. All user-facing, system output messages, **application log messages**, and technical messages **MUST** be externalized to resource bundles with type-safe message key access and comprehensive locale support.

**Required Components**:
- **Complete String Externalization**: All hardcoded strings converted to message keys in resource bundles
- **Type-Safe Message Keys**: Centralized constants class with organized message key categories
- **Localized Logging Service**: Internationalized wrapper for SLF4J loggers with locale context — **direct string logging is prohibited**
- **Bean Validation Integration**: Custom message interpolator for localized validation messages
- **Resource Bundle Extension**: Extended existing i18n infrastructure with new message categories
- **Configurable Default Locale**: Application uses `app.i18n.default-locale` property (configurable at deployment) with fallback to `Locale.getDefault()`

# Constraints:

- All message keys **MUST** follow dot-notation naming convention (e.g., `log.user.creating`, `validation.email.required`)
- Resource bundle files **MUST** exist for all supported locales: `en`, `es`, `fr`, `de`
- Placeholder count in translated messages **MUST** match the base (English) message for each key
- Message resolution **MUST NOT** introduce blocking I/O; bundles **MUST** be loaded and cached at startup via `ConcurrentHashMap`
- Direct string literals in logging calls (`LOG.infof(...)`, `logger.info(...)`) **MUST NOT** appear in application code; all logging **MUST** use `LocalizedLogger` with `MessageKeys` constants
- The default locale **MUST NOT** be hardcoded in application code; it **MUST** be configurable via `app.i18n.default-locale` property or `APP_I18N_DEFAULT_LOCALE` environment variable, falling back to `Locale.getDefault()`
- New message keys **MUST** be added to `MessageKeys.java` constants class before use; raw string keys **MUST NOT** be passed directly

# Alternatives:

**Option 1 — Database-Driven Messages**: Store all messages in database tables with locale columns. Rejected because every message resolution would require a database query (or a cache layer replicating what `ResourceBundle` already provides). Per the Java `ResourceBundle` Javadoc (https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ResourceBundle.html), `ResourceBundle.getBundle()` caches bundles by default, making file-based resolution effectively zero-cost after first load. A database approach adds network round-trip latency (~1–5 ms per query on local networks) and a runtime dependency that can fail independently of the application.

**Option 2 — External Translation Services (e.g., Google Cloud Translation API, AWS Translate)**: Use machine translation APIs for automatic resolution at runtime. Rejected because Google Cloud Translation API documentation (https://cloud.google.com/translate/quotas) documents a default rate limit of 600,000 characters/minute and introduces per-call network latency (50–200 ms per API call as documented in cloud service SLAs). This adds an external service dependency, cost per character, and a failure mode for every message resolution.

# Rationale:

Java `ResourceBundle` with `ConcurrentHashMap` caching (as implemented in `MessageResolver`) loads `.properties` files once at startup and serves subsequent lookups from memory. According to the Java 21 `ResourceBundle` Javadoc, the default caching strategy retains loaded bundles for the lifetime of the JVM, making per-lookup cost negligible (in-memory `HashMap` access). Type-safe message keys via the `MessageKeys` constants class eliminate runtime `MissingResourceException` errors caused by key typos — compile-time constant references ensure key validity.

**Why Prohibit Hardcoded Log Messages:**
- **Consistency**: Centralized message management ensures consistent terminology and formatting across all log output
- **Maintainability**: Changing a log message requires only updating the properties file, not searching through code
- **Auditability**: All possible log messages are documented in a single location for security and compliance review
- **Localization**: Operations teams in different regions can view logs in their preferred language
- **Searchability**: Log analysis tools can search by message key, enabling precise log queries regardless of language

**Why Use Parametric Default Locale:**
- **Deployment Flexibility**: Default locale can be changed per environment via `app.i18n.default-locale` property or `APP_I18N_DEFAULT_LOCALE` environment variable
- **Environment Respect**: Falls back to system's default locale (`Locale.getDefault()`) when not explicitly configured
- **Container Compatibility**: Kubernetes/Docker environments can set locale via environment variable in deployment manifests, per Quarkus Configuration Reference (https://quarkus.io/guides/config-reference)
- **Multi-Tenant Support**: Different deployments can serve different default languages without code changes
- **Zero Downtime Changes**: Locale can be changed via ConfigMap/Secrets without redeploying the application

# Implementation Guidelines:

## Mandatory Rules

1. **No Hardcoded Log Messages**: Log messages **MUST NOT** be written directly in application code. All log messages **MUST** be defined in properties files and referenced via message keys.
2. **No Hardcoded System Output Messages**: Any message reflected in system output (console, files, external systems) **MUST** be externalized to resource bundles.
3. **Configurable Default Locale**: The default localization language **MUST** be configurable via `app.i18n.default-locale`. If not configured, it falls back to `Locale.getDefault()`.
4. **Type-Safe Message Keys**: All message references **MUST** use type-safe constants from the `MessageKeys` class.

## Quick Reference: Correct vs Incorrect

| PROHIBITED | REQUIRED |
|------------|----------|
| `LOG.debugf("Creating user: %s", email)` | `localizedLogger.debug(LOG, MessageKeys.Log.USER_CREATING, email)` |
| `LOG.infof("User created: %s", id)` | `localizedLogger.info(LOG, MessageKeys.Log.USER_CREATED, id)` |
| `LOG.errorf(error, "Failed to save")` | `localizedLogger.error(LOG, MessageKeys.Log.SAVE_FAILED, error)` |
| `logger.info("Processing request")` | `localizedLogger.info(LOG, MessageKeys.Log.PROCESSING, ...)` |

## Resource Bundle Extension:

**Complete Message Categories** (extend existing `messages_en.properties`):
```properties
# Validation Messages - English
validation.email.required=Email is required
validation.email.invalid=Email must be valid
validation.email.size=Email must not exceed 255 characters
validation.firstName.required=First name is required
validation.firstName.size=First name must be between 1 and 100 characters
validation.lastName.required=Last name is required  
validation.lastName.size=Last name must be between 1 and 100 characters
validation.birthDate.required=Birth date is required
validation.birthDate.past=Birth date must be in the past

# Structured Logging Messages
log.user.creating=Creating user with email: {0}
log.user.created=Successfully created user with ID: {0}
log.user.retrieving.id=Retrieving user with ID: {0}
log.user.retrieving.all=Retrieving all users with pagination
log.user.updated=Successfully updated user with ID: {0}
log.user.deleted=Successfully deleted user with ID: {0}
log.user.notFound=User not found with ID: {0}
log.user.validationFailed=User data validation failed: {0}

# API Documentation Messages
api.user.create.summary=Create a new user account
api.user.create.description=Creates a new user with the provided information
api.user.get.summary=Retrieve user by ID  
api.user.getAll.summary=Retrieve all users with pagination
api.user.update.summary=Update existing user information
api.user.delete.summary=Delete user account
api.response.200.description=Operation completed successfully
api.response.201.description=Resource created successfully
api.response.400.description=Invalid request data provided
api.response.404.description=Requested resource was not found

# Entity String Representations
entity.user.toString=User{id={0}, email='{1}', firstName='{2}', lastName='{3}'}
entity.user.displayName=User: {0} {1} ({2})

# Domain Exception Messages (technical context only)
domain.user.emailExists.context=Email address already registered: {0}
domain.user.invalidAge.context=Invalid age for user registration: {0} years
domain.user.notFound.context=User entity not found with identifier: {0}
```

**Spanish Translation** (`messages_es.properties`):
```properties
# Mensajes de Validación - Español
validation.email.required=El correo electrónico es obligatorio
validation.email.invalid=El correo electrónico debe ser válido
validation.email.size=El correo electrónico no debe exceder 255 caracteres
validation.firstName.required=El nombre es obligatorio
validation.firstName.size=El nombre debe tener entre 1 y 100 caracteres
validation.lastName.required=El apellido es obligatorio
validation.lastName.size=El apellido debe tener entre 1 y 100 caracteres

# Mensajes de Registro Estructurado
log.user.creating=Creando usuario con correo: {0}
log.user.created=Usuario creado exitosamente con ID: {0}
log.user.retrieving.id=Recuperando usuario con ID: {0}
log.user.updated=Usuario actualizado exitosamente con ID: {0}
log.user.deleted=Usuario eliminado exitosamente con ID: {0}

# Mensajes de Documentación API
api.user.create.summary=Crear una nueva cuenta de usuario
api.user.create.description=Crea un nuevo usuario con la información proporcionada
api.user.get.summary=Recuperar usuario por ID
api.response.201.description=Recurso creado exitosamente
api.response.400.description=Datos de solicitud inválidos proporcionados
```

## Type-Safe Message Key Constants:

```java
public final class MessageKeys {
    private MessageKeys() {} // Utility class

    /**
     * Validation message keys for Bean Validation integration.
     */
    public static final class Validation {
        // Email validation keys
        public static final String EMAIL_REQUIRED = "validation.email.required";
        public static final String EMAIL_INVALID = "validation.email.invalid";
        public static final String EMAIL_SIZE = "validation.email.size";

        // Name validation keys
        public static final String FIRST_NAME_REQUIRED = "validation.firstName.required";
        public static final String FIRST_NAME_SIZE = "validation.firstName.size";
        public static final String LAST_NAME_REQUIRED = "validation.lastName.required";
        public static final String LAST_NAME_SIZE = "validation.lastName.size";

        // Date validation keys
        public static final String BIRTH_DATE_REQUIRED = "validation.birthDate.required";
        public static final String BIRTH_DATE_PAST = "validation.birthDate.past";
    }

    /**
     * Structured logging message keys for consistent log output.
     */
    public static final class Log {
        // User operation log keys
        public static final String USER_CREATING = "log.user.creating";
        public static final String USER_CREATED = "log.user.created";
        public static final String USER_RETRIEVING_ID = "log.user.retrieving.id";
        public static final String USER_RETRIEVING_ALL = "log.user.retrieving.all";
        public static final String USER_UPDATED = "log.user.updated";
        public static final String USER_DELETED = "log.user.deleted";

        // Error log keys
        public static final String USER_NOT_FOUND = "log.user.notFound";
        public static final String USER_VALIDATION_FAILED = "log.user.validationFailed";
    }

    /**
     * API documentation message keys for OpenAPI integration.
     */
    public static final class Api {
        // User API operation summaries
        public static final String USER_CREATE_SUMMARY = "api.user.create.summary";
        public static final String USER_CREATE_DESCRIPTION = "api.user.create.description";
        public static final String USER_GET_SUMMARY = "api.user.get.summary";
        public static final String USER_GET_ALL_SUMMARY = "api.user.getAll.summary";
        public static final String USER_UPDATE_SUMMARY = "api.user.update.summary";
        public static final String USER_DELETE_SUMMARY = "api.user.delete.summary";

        // Standard HTTP response descriptions
        public static final String RESPONSE_200_DESCRIPTION = "api.response.200.description";
        public static final String RESPONSE_201_DESCRIPTION = "api.response.201.description";
        public static final String RESPONSE_400_DESCRIPTION = "api.response.400.description";
        public static final String RESPONSE_404_DESCRIPTION = "api.response.404.description";
    }

    /**
     * Entity string representation keys for toString() methods.
     */
    public static final class Entity {
        // User entity representations
        public static final String USER_TO_STRING = "entity.user.toString";
        public static final String USER_DISPLAY_NAME = "entity.user.displayName";
    }

    /**
     * Domain exception technical context keys (not user-facing).
     */
    public static final class Domain {
        // User domain exception contexts
        public static final String USER_EMAIL_EXISTS_CONTEXT = "domain.user.emailExists.context";
        public static final String USER_INVALID_AGE_CONTEXT = "domain.user.invalidAge.context";
        public static final String USER_NOT_FOUND_CONTEXT = "domain.user.notFound.context";
    }
}
```

## Configurable Locale Context Provider:

```java
@ApplicationScoped
public class LocaleContextProvider {

    private final Locale configuredDefaultLocale;

    @Inject
    public LocaleContextProvider(
            @ConfigProperty(name = "app.i18n.default-locale", defaultValue = "") 
            Optional<String> defaultLocaleConfig) {
        
        this.configuredDefaultLocale = defaultLocaleConfig
            .filter(s -> !s.isBlank())
            .map(this::parseLocale)
            .orElse(null); // null means use Locale.getDefault()
    }

    /**
     * Get the current locale from request context, or fall back to configured/system default.
     */
    public Locale getCurrentLocale() {
        // Try to get from request context (e.g., Accept-Language header)
        return getRequestLocale()
            .orElseGet(this::getDefaultLocale);
    }

    /**
     * Get the configured default locale.
     * Falls back to system default if not explicitly configured.
     */
    public Locale getDefaultLocale() {
        return configuredDefaultLocale != null 
            ? configuredDefaultLocale 
            : Locale.getDefault();
    }

    /**
     * Check if a specific locale is configured (vs using system default).
     */
    public boolean hasConfiguredDefaultLocale() {
        return configuredDefaultLocale != null;
    }

    /**
     * Get locale from current request context (if available).
     */
    private Optional<Locale> getRequestLocale() {
        // Implementation depends on your request context mechanism
        // Example: from JAX-RS @Context HttpHeaders or Vert.x RoutingContext
        return Optional.empty(); // Override in subclass or inject RequestScoped provider
    }

    private Locale parseLocale(String localeString) {
        if (localeString.contains("_")) {
            String[] parts = localeString.split("_");
            return parts.length >= 2 
                ? new Locale(parts[0], parts[1])
                : new Locale(parts[0]);
        }
        if (localeString.contains("-")) {
            return Locale.forLanguageTag(localeString);
        }
        return new Locale(localeString);
    }
}
```

## Localized Logging Service:

```java
@ApplicationScoped
public class LocalizedLogger {

    @Inject
    private MessageResolver messageResolver;

    @Inject
    private LocaleContextProvider localeContextProvider;

    /**
     * Log debug message with localization support.
     */
    public void debug(Logger logger, String messageKey, Object... parameters) {
        if (logger.isDebugEnabled()) {
            String message = resolveMessage(messageKey, parameters);
            logger.debug(message);
        }
    }

    /**
     * Log info message with localization support.
     */
    public void info(Logger logger, String messageKey, Object... parameters) {
        if (logger.isInfoEnabled()) {
            String message = resolveMessage(messageKey, parameters);
            logger.info(message);
        }
    }

    /**
     * Log warning message with localization support.
     */
    public void warn(Logger logger, String messageKey, Object... parameters) {
        if (logger.isWarnEnabled()) {
            String message = resolveMessage(messageKey, parameters);
            logger.warn(message);
        }
    }

    /**
     * Log error message with localization support.
     */
    public void error(Logger logger, String messageKey, Object... parameters) {
        if (logger.isErrorEnabled()) {
            String message = resolveMessage(messageKey, parameters);
            logger.error(message);
        }
    }

    /**
     * Log error message with exception and localization support.
     */
    public void error(Logger logger, String messageKey, Throwable throwable, Object... parameters) {
        if (logger.isErrorEnabled()) {
            String message = resolveMessage(messageKey, parameters);
            logger.error(message, throwable);
        }
    }

    /**
     * Resolve message with current locale context.
     */
    private String resolveMessage(String messageKey, Object... parameters) {
        Locale currentLocale = localeContextProvider.getCurrentLocale();
        return messageResolver.getMessage(messageKey, currentLocale, parameters);
    }
}
```

## Bean Validation Integration:

**Localized Message Interpolator**:
```java
@ApplicationScoped
public class LocalizedMessageInterpolator implements MessageInterpolator {

    @Inject
    private MessageResolver messageResolver;

    @Inject
    private LocaleContextProvider localeContextProvider;

    @Override
    public String interpolate(String messageTemplate, Context context) {
        Locale locale = localeContextProvider.getCurrentLocale();
        return interpolate(messageTemplate, context, locale);
    }

    @Override
    public String interpolate(String messageTemplate, Context context, Locale locale) {
        // Handle message key format: {message.key}
        if (messageTemplate.startsWith("{") && messageTemplate.endsWith("}")) {
            String messageKey = messageTemplate.substring(1, messageTemplate.length() - 1);
            try {
                return messageResolver.getMessage(messageKey, locale);
            } catch (Exception e) {
                // Fallback to original template if resolution fails
                return messageTemplate;
            }
        }
        
        // Return template as-is if not a message key reference
        return messageTemplate;
    }
}
```

**Bean Validation Configuration**:
```java
@ApplicationScoped
public class ValidationConfig {

    @Inject
    private LocalizedMessageInterpolator localizedMessageInterpolator;

    /**
     * Provide localized message interpolator for Bean Validation.
     */
    @Produces
    @Dependent
    public MessageInterpolator messageInterpolator() {
        return localizedMessageInterpolator;
    }
}
```

## DTO Validation Integration:

**Localized Validation Annotations**:
```java
@RegisterForReflection
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

    public CreateUserRequest() {}
    
    public CreateUserRequest(String email, String firstName, String lastName) {
        this.email = email;
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // Domain command conversion
    public CreateUserCommand toCommand() {
        return CreateUserCommand.builder()
            .email(email)
            .firstName(firstName)
            .lastName(lastName)
            .build();
    }

    // Getters and setters
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
}
```

## Service Layer Integration:

### ❌ Anti-Pattern: Hardcoded Log Messages (PROHIBITED)

> **WARNING**: The following patterns are ALL PROHIBITED. Any code using these patterns MUST be rejected in code review.

```java
// DO NOT DO THIS - Hardcoded log messages are prohibited
@ApplicationScoped
public class BadUserService {
    // ❌ WRONG: SLF4J Logger with direct strings
    private static final Logger logger = LoggerFactory.getLogger(BadUserService.class);
    
    public Uni<User> createUser(CreateUserCommand command) {
        // ❌ WRONG: Hardcoded log message
        logger.info("Creating user with email: " + command.getEmail());
        
        // ❌ WRONG: String concatenation in log
        logger.debug("User " + command.getFirstName() + " validated successfully");
        
        // ❌ WRONG: Formatted string directly
        logger.error(String.format("Failed to create user: %s", command.getEmail()));
    }
}
```

```java
// DO NOT DO THIS - JBoss logging with formatted strings is also prohibited
@ApplicationScoped
public class BadAuditRepository {
    // ❌ WRONG: JBoss Logger with direct formatted strings
    private static final org.jboss.logging.Logger LOG = 
        org.jboss.logging.Logger.getLogger(BadAuditRepository.class);
    
    public Uni<Void> save(AuditEvent event) {
        // ❌ WRONG: LOG.debugf with hardcoded format string
        LOG.debugf("Saving audit event: %s", event.id());
        
        // ❌ WRONG: LOG.infof with hardcoded message
        LOG.infof("Saved %d events successfully", count);
        
        // ❌ WRONG: LOG.errorf with hardcoded error message
        LOG.errorf(error, "Failed to save audit event: %s", event.id());
        
        // ❌ WRONG: Even simple messages are prohibited
        LOG.debug("Connection acquired");
        LOG.info("Batch processing started");
    }
}
```

**Why these are prohibited:**
- Messages cannot be translated for international operations teams
- No centralized message management for consistency
- Log analysis tools cannot search by message key
- Security/compliance audits require documented message catalog

### ✅ Correct Pattern: Externalized Log Messages
**Updated Service with Localized Logging**:
```java
@ApplicationScoped
public class UserService {

    private static final Logger logger = LoggerFactory.getLogger(UserService.class);

    @Inject
    private LocalizedLogger localizedLogger;

    @Inject
    private UserRepository userRepository;

    /**
     * Create new user with localized logging.
     */
    public Uni<User> createUser(CreateUserCommand command) {
        localizedLogger.debug(logger, MessageKeys.Log.USER_CREATING, command.getEmail());

        return userRepository.findByEmail(command.getEmail())
            .onItem().ifNotNull().failWith(() -> {
                localizedLogger.warn(logger, MessageKeys.Log.USER_VALIDATION_FAILED, 
                    "Email already exists: " + command.getEmail());
                return new EmailAlreadyExistsException(command.getEmail());
            })
            .onItem().ifNull().switchTo(() -> {
                User newUser = User.create(
                    command.getEmail(),
                    command.getFirstName(),
                    command.getLastName()
                );
                
                return userRepository.save(newUser)
                    .invoke(savedUser -> {
                        localizedLogger.info(logger, MessageKeys.Log.USER_CREATED, savedUser.getId());
                    });
            });
    }

    /**
     * Retrieve user by ID with localized logging.
     */
    public Uni<User> findById(String userId) {
        localizedLogger.debug(logger, MessageKeys.Log.USER_RETRIEVING_ID, userId);

        return userRepository.findById(userId)
            .onItem().ifNull().failWith(() -> {
                localizedLogger.warn(logger, MessageKeys.Log.USER_NOT_FOUND, userId);
                return new UserNotFoundException(userId);
            });
    }

    /**
     * Update existing user with localized logging.
     */
    public Uni<User> updateUser(UpdateUserCommand command) {
        return findById(command.getUserId())
            .onItem().transform(user -> {
                if (command.hasFirstName()) {
                    user.updateFirstName(command.getFirstName());
                }
                if (command.hasLastName()) {
                    user.updateLastName(command.getLastName());
                }
                return user;
            })
            .chain(userRepository::save)
            .invoke(updatedUser -> {
                localizedLogger.info(logger, MessageKeys.Log.USER_UPDATED, updatedUser.getId());
            });
    }
}
```

## Enhanced MessageResolver Integration:

```java
@ApplicationScoped
public class MessageResolver {

    private final ConcurrentHashMap<String, ResourceBundle> bundleCache = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, MessageFormat> formatCache = new ConcurrentHashMap<>();

    @Inject
    private LocaleContextProvider localeContextProvider;

    /**
     * Get simple message for current locale.
     */
    public String getMessage(String messageKey, Object... parameters) {
        Locale currentLocale = localeContextProvider.getCurrentLocale();
        return getMessage(messageKey, currentLocale, parameters);
    }

    /**
     * Get message for specific locale.
     * Falls back to configured default locale (or system default), then to message key if not found.
     */
    public String getMessage(String messageKey, Locale locale, Object... parameters) {
        try {
            ResourceBundle bundle = getResourceBundle(locale);
            return formatMessage(bundle, messageKey, locale, parameters);
        } catch (MissingResourceException e) {
            // Fallback to configured default locale (or system default if not configured)
            Locale fallbackLocale = localeContextProvider.getDefaultLocale();
            if (!fallbackLocale.equals(locale)) {
                return getMessage(messageKey, fallbackLocale, parameters);
            }
            
            // Ultimate fallback to key with parameters
            return createFallbackMessage(messageKey, parameters);
        }
    }

    /**
     * Get structured localized message (for complex error responses).
     * Falls back to system default locale if requested locale bundle not found.
     */
    public LocalizedMessage resolve(String errorCode, Locale locale, Object... parameters) {
        try {
            ResourceBundle bundle = getResourceBundle(locale);
            
            String title = formatMessage(bundle, errorCode + ".title", locale, parameters);
            String detail = formatMessage(bundle, errorCode + ".detail", locale, parameters);
            String action = formatMessage(bundle, errorCode + ".action", locale, parameters);
            
            return LocalizedMessage.builder()
                .errorCode(errorCode)
                .locale(locale)
                .title(title)
                .detail(detail)
                .action(action)
                .build();
                
        } catch (MissingResourceException e) {
            // Fallback to configured default locale (or system default if not configured)
            Locale fallbackLocale = localeContextProvider.getDefaultLocale();
            if (!fallbackLocale.equals(locale)) {
                return resolve(errorCode, fallbackLocale, parameters);
            }
            
            return createFallbackLocalizedMessage(errorCode, locale);
        }
    }

    private ResourceBundle getResourceBundle(Locale locale) {
        String key = locale.toLanguageTag();
        return bundleCache.computeIfAbsent(key, k -> {
            return ResourceBundle.getBundle("i18n.messages", locale, 
                Thread.currentThread().getContextClassLoader());
        });
    }

    private String formatMessage(ResourceBundle bundle, String key, Locale locale, Object... parameters) {
        try {
            String pattern = bundle.getString(key);
            if (parameters.length == 0) {
                return pattern;
            }
            
            String formatKey = locale.toLanguageTag() + ":" + key;
            MessageFormat format = formatCache.computeIfAbsent(formatKey, 
                k -> new MessageFormat(pattern, locale));
            
            return format.format(parameters);
            
        } catch (MissingResourceException e) {
            throw e; // Re-throw for fallback handling
        }
    }

    private String createFallbackMessage(String messageKey, Object... parameters) {
        if (parameters.length == 0) {
            return String.format("[%s]", messageKey);
        }
        return String.format("[%s: %s]", messageKey, String.join(", ", 
            java.util.Arrays.stream(parameters).map(String::valueOf).toArray(String[]::new)));
    }

    private LocalizedMessage createFallbackLocalizedMessage(String errorCode, Locale locale) {
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

## Testing Strategy:

**Message Bundle Completeness Test**:
```java
@QuarkusTest
class MessageBundleCompletenessTest {

    private static final List<Locale> SUPPORTED_LOCALES = List.of(
        Locale.ENGLISH, new Locale("es"), Locale.FRENCH, Locale.GERMAN
    );

    @Test
    void shouldHaveAllMessageKeysInAllLocales() {
        ResourceBundle baseBundle = ResourceBundle.getBundle("messages", Locale.ENGLISH);
        Set<String> baseKeys = baseBundle.keySet();

        for (Locale locale : SUPPORTED_LOCALES) {
            ResourceBundle localeBundle = ResourceBundle.getBundle("messages", locale);
            Set<String> missingKeys = baseKeys.stream()
                .filter(key -> !localeBundle.containsKey(key))
                .collect(Collectors.toSet());
            
            assertThat(missingKeys)
                .as("Missing keys in locale %s", locale)
                .isEmpty();
        }
    }

    @Test
    void shouldHaveConsistentPlaceholderCountAcrossLocales() {
        ResourceBundle baseBundle = ResourceBundle.getBundle("messages", Locale.ENGLISH);
        
        for (String key : baseBundle.keySet()) {
            int basePlaceholders = countPlaceholders(baseBundle.getString(key));
            
            for (Locale locale : SUPPORTED_LOCALES) {
                ResourceBundle localeBundle = ResourceBundle.getBundle("messages", locale);
                if (localeBundle.containsKey(key)) {
                    int localePlaceholders = countPlaceholders(localeBundle.getString(key));
                    assertThat(localePlaceholders)
                        .as("Placeholder count mismatch for key '%s' in locale %s", key, locale)
                        .isEqualTo(basePlaceholders);
                }
            }
        }
    }

    private int countPlaceholders(String message) {
        return (int) Pattern.compile("\\{\\d+\\}").matcher(message).results().count();
    }
}
```

**Localized Validation Test**:
```java
@QuarkusTest
class LocalizedValidationTest {

    @Inject
    Validator validator;

    @Test
    void shouldReturnSpanishValidationMessages() {
        CreateUserRequest request = new CreateUserRequest("", "invalid-email", null);
        
        Set<ConstraintViolation<CreateUserRequest>> violations = validator.validate(request);
        
        // Verify messages are resolved from Spanish bundle
        Locale.setDefault(new Locale("es"));
        assertThat(violations)
            .extracting(ConstraintViolation::getMessage)
            .anyMatch(msg -> msg.contains("obligatorio"));
    }
}
```

**Localized Logging Test**:
```java
@QuarkusTest
class LocalizedLoggerTest {

    @Inject
    LocalizedLogger localizedLogger;

    @Test
    void shouldLogWithLocalizedMessage() {
        UUID userId = UUID.randomUUID();
        
        // Should resolve message from bundle with parameter substitution
        localizedLogger.info(MessageKeys.Log.USER_CREATED, userId);
        
        // Verify log output contains formatted message
    }
}
```

## Configuration and Validation:

**Quarkus Configuration** (`application.properties`):
```properties
# Message Externalization Configuration
quarkus.locales=en,es,fr,de

# =============================================================================
# CONFIGURABLE DEFAULT LOCALE
# =============================================================================
# Default locale for message resolution. Can be overridden at deployment time.
# If not set, falls back to system default locale (Locale.getDefault()).
#
# Supported formats:
#   - Language only: en, es, fr, de
#   - Language + Country: en_US, es_ES, fr_FR, de_DE
#   - BCP 47 tag: en-US, es-ES, fr-FR, de-DE
#
# Leave empty or comment out to use system default locale.
# app.i18n.default-locale=en

# Override via environment variable for different deployments:
# APP_I18N_DEFAULT_LOCALE=es_ES  (for Spanish deployment)
# APP_I18N_DEFAULT_LOCALE=de_DE  (for German deployment)
# =============================================================================

# Resource bundle configuration for GraalVM native compilation
quarkus.native.resources.includes=i18n/**/*.properties

# Bean Validation configuration
quarkus.hibernate-validator.method-validation.allow-parameter-constraints=true
quarkus.hibernate-validator.method-validation.allow-multiple-cascaded-validation=true

# Logging configuration for structured output
quarkus.log.level=INFO
quarkus.log.console.json=true
quarkus.log.console.json.pretty-print=false
```

**Deployment Configuration Examples:**

*Docker Compose (docker-compose.yml):*
```yaml
services:
  app:
    image: myapp:latest
    environment:
      APP_I18N_DEFAULT_LOCALE: "es_ES"  # Spanish for this deployment
```

*Kubernetes Deployment:*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: app
          env:
            - name: APP_I18N_DEFAULT_LOCALE
              value: "de_DE"  # German for this deployment
            # Or from ConfigMap:
            - name: APP_I18N_DEFAULT_LOCALE
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: default-locale
```

*Kubernetes ConfigMap for Multi-Region:*
```yaml
# configmap-us.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  default-locale: "en_US"
---
# configmap-latam.yaml  
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  default-locale: "es_ES"
---
# configmap-eu.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  default-locale: "de_DE"
```

*Command Line Override:*
```bash
# Run with Spanish locale
java -Dapp.i18n.default-locale=es_ES -jar myapp.jar

# Or via environment variable
APP_I18N_DEFAULT_LOCALE=fr_FR java -jar myapp.jar
```

**Validation Commands:
```bash
# Test message externalization with different locales
./mvnw test -Dtest=MessageKeysValidationTest

# Test localized logging service
./mvnw test -Dtest=LocalizedLoggerTest

# Test Bean Validation message interpolation
./mvnw test -Dtest=LocalizedMessageInterpolatorTest

# Test service layer with localized messages
./mvnw test -Dtest=UserServiceLocalizationTest

# Verify all message keys have translations
./mvnw test -Dtest=MessageBundleCompletenessTest

# Test REST endpoints with localized validation
curl -H "Accept-Language: es-ES" -H "Content-Type: application/json" \
     -X POST "http://localhost:8080/api/v1/users" \
     -d '{"email":"invalid","firstName":"","lastName":""}'

# Test query parameter locale override
curl -H "Content-Type: application/json" \
     -X POST "http://localhost:8080/api/v1/users?lang=fr" \
     -d '{"email":"invalid","firstName":"","lastName":""}'

# Verify GraalVM native compilation with externalized messages
# Verify GraalVM native compilation with externalized messages
./mvnw package -Pnative -DskipTests
```

# Additional Recommendations:

- Cache loaded bundles per locale with `ConcurrentHashMap` for thread-safe operation
- Create IDE live templates/snippets for common message key patterns
- Implement automated detection of missing translations during build process
- Log fallback message usage to identify missing translations
- Gradually migrate hardcoded strings to minimize risk

**References:**
- Java 21 `ResourceBundle` Javadoc: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ResourceBundle.html
- Java 21 `MessageFormat` Javadoc: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/text/MessageFormat.html
- Jakarta Bean Validation 3.0 Specification (Message Interpolation): https://jakarta.ee/specifications/bean-validation/3.0/
- Quarkus Localization Guide: https://quarkus.io/guides/localization
- Quarkus Configuration Reference: https://quarkus.io/guides/config-reference

## Code Review Checklist for Message Externalization

| Check | How to Verify | Action if Violated |
|-------|--------------|-------------------|
| No `LOG.debugf(...)`, `LOG.infof(...)`, `LOG.errorf(...)` | Search for `LOG\.(debug\|info\|warn\|error\|trace)f?\(` | **REJECT PR** |
| No `logger.info(...)`, `logger.debug(...)`, etc. with strings | Search for `logger\.(info\|debug\|warn\|error\|trace)\("` | **REJECT PR** |
| Uses `LocalizedLogger` injection | Verify `@Inject LocalizedLogger localizedLogger` | **REJECT PR** |
| Uses `MessageKeys.Log.*` constants | Verify `localizedLogger.*(LOG, MessageKeys.Log.*, ...)` | **REJECT PR** |
| Message keys defined in `MessageKeys.java` | Check `MessageKeys.Log` inner class | **REJECT PR** |
| Translations in ALL 4 locales | Check `messages_en.properties`, `_es`, `_fr`, `_de` | **REJECT PR** |
| New features include logging messages | Verify log messages for operations | Request additions |

## Git Pre-Commit Hook (Optional)

Add to `.git/hooks/pre-commit`:
```bash
#!/bin/bash
# Prevent commits with hardcoded log messages

VIOLATIONS=$(git diff --cached --name-only | xargs grep -l '\.java$' 2>/dev/null | \
  xargs grep -n 'LOG\.\(debug\|info\|warn\|error\|trace\)f\?(' 2>/dev/null | \
  grep -v 'LocalizedLogger' | wc -l)

if [ "$VIOLATIONS" -gt 0 ]; then
  echo "COMMIT BLOCKED: Hardcoded log messages detected"
  echo "   Use LocalizedLogger with MessageKeys instead."
  git diff --cached --name-only | xargs grep -l '\.java$' 2>/dev/null | \
    xargs grep -n 'LOG\.\(debug\|info\|warn\|error\|trace\)f\?(' 2>/dev/null | \
    grep -v 'LocalizedLogger' | head -10
  exit 1
fi
```

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| No Hardcoded Strings | `grep -rn 'return ".*[a-zA-Z].*"' */src/main/java/**/api` | Zero hardcoded user-facing strings in API layer |
| No Hardcoded Log Messages | `grep -rn 'LOG\.\(debug\|info\|warn\|error\|trace\)f\?(' */src/main/java` | Zero direct string literals in log calls |
| Message Keys | `grep -r "MessageKeys" */src/main/java` | Type-safe constants used for all message references |
| Error Messages | `find . -name "messages*.properties" -path "*/resources/*"` | All errors externalized in bundles |
| Log Messages | `grep -r "LocalizedLogger" */src/main/java` | Log messages use `LocalizedLogger` with keys only |
| Validation Messages | `grep -r 'message\s*=\s*"{' */src/main/java` | Custom message keys in DTO annotations |
| Message Format | `grep -r "^[a-z][a-z.]*=" */src/main/resources/*.properties` | Dot-notation naming convention |
| System Default Locale | `grep -r "app.i18n.default-locale" */src/main/resources/application*.properties` | No hardcoded default locale |
| Externalization Tests | `./mvnw test -Dtest=*MessageBundle*,*LocalizedLogger*,*Interpolator*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Message Externalization..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for message properties files
BUNDLES=$(find . -name "messages*.properties" -path "*/resources/*" 2>/dev/null | wc -l)
if [ "$BUNDLES" -gt 0 ]; then
  echo "✅ Message bundles found: $BUNDLES"
else
  echo "❌ No message bundles found"
  FAIL=1
fi

# Check for multiple locales (en, es, fr, de required)
LOCALE_COUNT=$(find . -name "messages_*.properties" -path "*/resources/*" 2>/dev/null | sed 's/.*messages_\(.*\)\.properties/\1/' | sort -u | wc -l)
if [ "$LOCALE_COUNT" -ge 4 ]; then
  echo "✅ All required locales present: $LOCALE_COUNT"
else
  echo "❌ Fewer than 4 locale bundles found (need en, es, fr, de)"
  FAIL=1
fi

# Check for hardcoded log messages (SLF4J)
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq 'logger\.\(info\|debug\|warn\|error\|trace\)("' "$sp" 2>/dev/null | grep -v "LocalizedLogger" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 0 ]; then
  echo "✅ No hardcoded SLF4J log messages found"
else
  echo "❌ Hardcoded SLF4J log messages detected"
  FAIL=1
fi

# Check for hardcoded log messages (JBoss LOG)
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq 'LOG\.\(info\|debug\|warn\|error\|trace\)f\?(' "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 0 ]; then
  echo "✅ No hardcoded JBoss LOG messages found"
else
  echo "❌ Hardcoded JBoss LOG messages detected"
  FAIL=1
fi

# Check for LocalizedLogger usage
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "LocalizedLogger" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ LocalizedLogger usage found"
else
  echo "❌ No LocalizedLogger usage found"
  FAIL=1
fi

# Check for MessageKeys constants
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "MessageKeys" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ MessageKeys constants found"
else
  echo "❌ No MessageKeys constants found"
  FAIL=1
fi

# Check for externalized validation messages in DTOs
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq 'message\s*=\s*"{' "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Validation messages use externalized keys"
else
  echo "❌ Validation messages not externalized"
  FAIL=1
fi

# Check that default locale is not hardcoded
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq 'quarkus.default-locale=' "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 0 ]; then
  echo "✅ Default locale not hardcoded"
else
  echo "❌ Default locale is hardcoded in application properties"
  FAIL=1
fi

# Check message key naming convention (dot notation)
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "^[a-z][a-z.]*=" "$rp"/*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Message keys follow dot-notation convention"
else
  echo "❌ Message keys do not follow naming convention"
  FAIL=1
fi

# Run externalization tests
if ./mvnw test -Dtest="*MessageBundle*,*LocalizedLogger*,*Interpolator*" -q 2>/dev/null; then
  echo "✅ Message externalization tests pass"
else
  echo "❌ Message externalization tests failed or not found"
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
  echo "❌ Message Externalization validation failed"
  exit 1
fi
echo "✅ All Message Externalization criteria met"
```
