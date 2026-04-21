# Issue: API Documentation Localization for OpenAPI

OpenAPI specifications rely on hardcoded English strings in annotations, preventing international development teams from accessing localized API documentation. This creates integration barriers, reduces API adoption in non-English markets, and complicates maintenance.

# Decision:

Implement hybrid OpenAPI documentation combining static build-time documentation with runtime localization capabilities. OpenAPI annotations **MUST** maintain static English content for tool compatibility, while dedicated runtime endpoints **MUST** provide localized documentation through existing message resolution infrastructure.

**Required Components**:
- Static OpenAPI annotations with message key comments for future automation
- Runtime `ApiDocumentationService` for dynamic localized documentation
- Extended `MessageKeys.Api` constants for type-safe documentation keys  
- Complete resource bundle translations for all API documentation strings
- Localized documentation endpoints accessible via REST API

# Constraints:

- Static annotations **MUST** remain unchanged to preserve toolchain compatibility
- Build-time documentation **MUST** continue working without runtime dependencies
- Runtime documentation resolution **MUST NOT** exceed 10ms response time overhead
- Memory overhead **MUST NOT** exceed 5MB per supported locale
- Documentation localization **MUST** remain in infrastructure layer
- All components **MUST** be thread-safe for reactive concurrent access
- **MUST** be GraalVM native compilation compatible
- **MUST** gracefully degrade to English when requested locale is unavailable

# Alternatives:

**Option 1 — Dynamic OpenAPI Annotation Localization**: Modify annotations at runtime to inject localized strings. Rejected because the OpenAPI Specification v3.0.3 (https://spec.openapis.org/oas/v3.0.3) defines `summary` and `description` as static string fields, and SmallRye OpenAPI (https://github.com/smallrye/smallrye-open-api) processes annotations at build time via annotation processors. Runtime modification would require bytecode manipulation that conflicts with GraalVM native-image ahead-of-time compilation, which does not support runtime class redefinition per GraalVM documentation (https://www.graalvm.org/latest/reference-manual/native-image/metadata/Compatibility/).

**Option 2 — Separate OpenAPI Documents Per Locale**: Maintain individual OpenAPI specification files for each supported language. Rejected because each new API endpoint or schema change would require synchronized updates across 4+ locale-specific files. The OpenAPI Specification provides no built-in multi-locale support or inheritance mechanism, meaning each file is a full duplicate (~500+ lines per spec), creating N-fold maintenance cost where N is the number of supported locales.

# Rationale:

The OpenAPI Specification v3.0.3 (https://spec.openapis.org/oas/v3.0.3) defines `summary` and `description` as plain string fields with no built-in localization mechanism. SmallRye OpenAPI (used by Quarkus per https://quarkus.io/guides/openapi-swaggerui) processes `@Operation` annotations at build time, producing a static `openapi.json`/`openapi.yaml`. Maintaining static English annotations preserves compatibility with this toolchain and with downstream consumers (Swagger UI, code generators).

The runtime `ApiDocumentationService` reuses the existing `MessageResolver` with `ConcurrentHashMap` caching (as defined in ADR-26), which loads `.properties` bundles once and serves subsequent lookups from in-memory cache. Per the Java 21 `ResourceBundle` Javadoc (https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ResourceBundle.html), cached bundles are retained for the lifetime of the JVM, keeping per-request documentation resolution to in-memory map access. Message key comments in static annotations (`// MessageKeys.Api.USER_CREATE_SUMMARY`) provide a traceable link between static and runtime documentation for future automation.

# Implementation Guidelines:

## Message Key Infrastructure:

**MessageKeys.Api Extension**:
```java
public final class MessageKeys {
    public static final class Api {
        // User API operations
        public static final String USER_CREATE_SUMMARY = "api.user.create.summary";
        public static final String USER_CREATE_DESCRIPTION = "api.user.create.description";
        public static final String USER_GET_SUMMARY = "api.user.get.summary";
        public static final String USER_GET_DESCRIPTION = "api.user.get.description";
        public static final String USER_GET_ALL_SUMMARY = "api.user.getAll.summary";
        public static final String USER_GET_ALL_DESCRIPTION = "api.user.getAll.description";
        public static final String USER_UPDATE_SUMMARY = "api.user.update.summary";
        public static final String USER_UPDATE_DESCRIPTION = "api.user.update.description";
        public static final String USER_DELETE_SUMMARY = "api.user.delete.summary";
        public static final String USER_DELETE_DESCRIPTION = "api.user.delete.description";

        // Authentication API operations
        public static final String AUTH_ADMIN_TOKEN_SUMMARY = "api.auth.admin.token.summary";
        public static final String AUTH_ADMIN_TOKEN_DESCRIPTION = "api.auth.admin.token.description";
        public static final String AUTH_USER_TOKEN_SUMMARY = "api.auth.user.token.summary";
        public static final String AUTH_USER_TOKEN_DESCRIPTION = "api.auth.user.token.description";

        // Parameters and tags
        public static final String PARAM_USER_ID_DESCRIPTION = "api.param.userId.description";
        public static final String PARAM_ROLE_DESCRIPTION = "api.param.role.description";
        public static final String TAG_USERS_NAME = "api.tag.users.name";
        public static final String TAG_USERS_DESCRIPTION = "api.tag.users.description";
        public static final String TAG_AUTH_NAME = "api.tag.auth.name";
        public static final String TAG_AUTH_DESCRIPTION = "api.tag.auth.description";
    }
}
```

## Resource Bundle Extensions:

**English** (`messages_en.properties`):
```properties
# User API Documentation
api.user.create.summary=Create a new user
api.user.create.description=Creates a new user with the provided information
api.user.get.summary=Retrieve user by ID
api.user.get.description=Retrieves a user by their unique identifier
api.user.getAll.summary=Retrieve all users
api.user.getAll.description=Retrieves all users with optional pagination
api.user.update.summary=Update existing user
api.user.update.description=Updates an existing user with new information
api.user.delete.summary=Delete user account
api.user.delete.description=Removes a user from the system

# Authentication API Documentation
api.auth.admin.token.summary=Generate admin JWT token
api.auth.admin.token.description=Creates a JWT token with administrative privileges
api.auth.user.token.summary=Generate user JWT token
api.auth.user.token.description=Creates a JWT token with standard user privileges

# Parameters and Tags
api.param.userId.description=User unique identifier (UUID format)
api.param.role.description=User authorization role (ADMIN, USER, CUSTOM)
api.tag.users.name=Users
api.tag.users.description=User account management operations
api.tag.auth.name=Authentication
api.tag.auth.description=JWT token generation and authentication operations
```

**Spanish** (`messages_es.properties`):
```properties
# Documentación API de Usuario
api.user.create.summary=Crear un nuevo usuario
api.user.create.description=Crea un nuevo usuario con la información proporcionada
api.user.get.summary=Recuperar usuario por ID
api.user.get.description=Recupera un usuario por su identificador único
api.user.getAll.summary=Recuperar todos los usuarios
api.user.getAll.description=Recupera todos los usuarios con paginación opcional
api.user.update.summary=Actualizar usuario existente
api.user.update.description=Actualiza un usuario existente con nueva información
api.user.delete.summary=Eliminar cuenta de usuario
api.user.delete.description=Elimina un usuario del sistema

# Documentación API de Autenticación
api.auth.admin.token.summary=Generar token JWT de administrador
api.auth.admin.token.description=Crea un token JWT con privilegios administrativos
api.auth.user.token.summary=Generar token JWT de usuario
api.auth.user.token.description=Crea un token JWT con privilegios estándar de usuario

# Parámetros y Etiquetas
api.param.userId.description=Identificador único de usuario (formato UUID)
api.param.role.description=Rol de autorización de usuario (ADMIN, USER, CUSTOM)
api.tag.users.name=Usuarios
api.tag.users.description=Operaciones de gestión de cuentas de usuario
api.tag.auth.name=Autenticación
api.tag.auth.description=Operaciones de generación de tokens JWT y autenticación
```

## ApiDocumentationService:
```java
@ApplicationScoped
public class ApiDocumentationService {
    private final MessageResolver messageResolver;
    private final LocaleContextProvider localeContextProvider;
    private final Map<String, Map<String, String>> documentationCache = new ConcurrentHashMap<>();

    @Inject
    public ApiDocumentationService(MessageResolver messageResolver,
                                   LocaleContextProvider localeContextProvider) {
        this.messageResolver = messageResolver;
        this.localeContextProvider = localeContextProvider;
    }

    public String getLocalizedApiMessage(String messageKey) {
        Locale currentLocale = localeContextProvider.getCurrentLocale();
        return messageResolver.getMessage(messageKey, currentLocale);
    }

    public Map<String, Object> getLocalizedDocumentation() {
        Locale currentLocale = localeContextProvider.getCurrentLocale();
        return getLocalizedDocumentation(currentLocale);
    }

    public Map<String, Object> getLocalizedDocumentation(Locale locale) {
        String cacheKey = locale.toLanguageTag();
        return documentationCache.computeIfAbsent(cacheKey, k -> buildDocumentationMap(locale));
    }

    private Map<String, Object> buildDocumentationMap(Locale locale) {
        return Map.of(
            "users", Map.of(
                "create", Map.of(
                    "summary", messageResolver.getMessage(MessageKeys.Api.USER_CREATE_SUMMARY, locale),
                    "description", messageResolver.getMessage(MessageKeys.Api.USER_CREATE_DESCRIPTION, locale)
                ),
                "get", Map.of(
                    "summary", messageResolver.getMessage(MessageKeys.Api.USER_GET_SUMMARY, locale),
                    "description", messageResolver.getMessage(MessageKeys.Api.USER_GET_DESCRIPTION, locale)
                )
                // ... other operations
            ),
            "authentication", Map.of(
                "adminToken", Map.of(
                    "summary", messageResolver.getMessage(MessageKeys.Api.AUTH_ADMIN_TOKEN_SUMMARY, locale),
                    "description", messageResolver.getMessage(MessageKeys.Api.AUTH_ADMIN_TOKEN_DESCRIPTION, locale)
                )
                // ... other operations
            )
        );
    }
}
```

## Controller Integration:
```java
@Path("/api/v1/users")
@Tag(name = "Users", description = "User management operations") // MessageKeys.Api.TAG_USERS_NAME, TAG_USERS_DESCRIPTION
public class UserController {

    @Inject
    private UserService userService;

    @Inject
    private ApiDocumentationService apiDocumentationService;

    @POST
    @Operation(
        summary = "Create a new user", // MessageKeys.Api.USER_CREATE_SUMMARY
        description = "Creates a new user with the provided information") // MessageKeys.Api.USER_CREATE_DESCRIPTION
    @RolesAllowed({"ADMIN"})
    public Uni<Response> createUser(@Valid CreateUserRequest request) {
        return userService.createUser(request.toCommand())
            .map(user -> Response.status(Response.Status.CREATED)
                .entity(UserResponse.from(user))
                .build());
    }

    @GET
    @Path("/{id}")
    @Operation(
        summary = "Retrieve user by ID", // MessageKeys.Api.USER_GET_SUMMARY  
        description = "Retrieves a user by their unique identifier") // MessageKeys.Api.USER_GET_DESCRIPTION
    @RolesAllowed({"ADMIN", "USER"})
    public Uni<UserResponse> getUserById(
            @Parameter(description = "User unique identifier") // MessageKeys.Api.PARAM_USER_ID_DESCRIPTION
            @PathParam("id") String userId) {
        return userService.findById(userId)
            .map(UserResponse::from);
    }

    /**
     * Runtime localized documentation endpoint.
     */
    @GET
    @Path("/api-info")
    @Operation(
        summary = "Get localized API documentation",
        description = "Returns API documentation in the user's preferred language")
    @RolesAllowed({"ADMIN", "USER"})
    public Uni<ApiDocumentationResponse> getLocalizedApiDocumentation() {
        return Uni.createFrom().item(() -> {
            Map<String, Object> documentation = apiDocumentationService.getLocalizedDocumentation();
            return new ApiDocumentationResponse(documentation);
        });
    }
}
```

## Documentation Response DTO:
```java
@RegisterForReflection
public class ApiDocumentationResponse {
    @JsonProperty("documentation")
    private Map<String, Object> documentation;

    @JsonProperty("locale")
    private String locale;

    @JsonProperty("timestamp")
    private long timestamp;

    public ApiDocumentationResponse() {}

    public ApiDocumentationResponse(Map<String, Object> documentation) {
        this.documentation = documentation;
        this.locale = java.util.Locale.getDefault().toLanguageTag();
        this.timestamp = System.currentTimeMillis();
    }

    // Getters and setters
    public Map<String, Object> getDocumentation() { return documentation; }
    public void setDocumentation(Map<String, Object> documentation) { this.documentation = documentation; }
    public String getLocale() { return locale; }
    public void setLocale(String locale) { this.locale = locale; }
    public long getTimestamp() { return timestamp; }
    public void setTimestamp(long timestamp) { this.timestamp = timestamp; }
}
```

## Validation Commands:
```bash
# Test English documentation (default)
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
     -H "Accept-Language: en" \
     "http://localhost:8080/api/v1/users/api-info"

# Test Spanish documentation
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
     -H "Accept-Language: es" \
     "http://localhost:8080/api/v1/users/api-info"

# Test French documentation
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
     -H "Accept-Language: fr" \
     "http://localhost:8080/api/v1/users/api-info"

# Verify static OpenAPI documentation still works
curl "http://localhost:8080/q/openapi"

# Test Swagger UI accessibility
curl "http://localhost:8080/q/swagger-ui/"

# Run unit tests
../mvnw test -Dtest=ApiDocumentationServiceTest
../mvnw test -Dtest=UserControllerDocumentationTest
```

## Testing Strategy:

**API Documentation Localization Test**:
```java
@QuarkusTest
class ApiDocumentationLocalizationTest {

    private static final List<Locale> SUPPORTED_LOCALES = List.of(
        Locale.ENGLISH, new Locale("es"), Locale.FRENCH, Locale.GERMAN
    );

    @Inject
    ApiDocumentationService documentationService;

    @Test
    void shouldHaveAllApiKeysInAllLocales() {
        ResourceBundle baseBundle = ResourceBundle.getBundle("messages", Locale.ENGLISH);
        Set<String> apiKeys = baseBundle.keySet().stream()
            .filter(key -> key.startsWith("api."))
            .collect(Collectors.toSet());

        for (Locale locale : SUPPORTED_LOCALES) {
            ResourceBundle localeBundle = ResourceBundle.getBundle("messages", locale);
            Set<String> missingKeys = apiKeys.stream()
                .filter(key -> !localeBundle.containsKey(key))
                .collect(Collectors.toSet());
            
            assertThat(missingKeys)
                .as("Missing API keys in locale %s", locale)
                .isEmpty();
        }
    }

    @Test
    void shouldReturnConsistentDocumentationStructure() {
        for (Locale locale : SUPPORTED_LOCALES) {
            Map<String, Object> docs = documentationService.getDocumentation(locale);
            
            assertThat(docs).containsKeys("operations", "tags", "parameters");
            assertThat(docs.get("operations")).isInstanceOf(Map.class);
        }
    }

    @Test
    void shouldFallbackToEnglishForUnsupportedLocales() {
        Locale unsupported = new Locale("xx");
        Map<String, Object> docs = documentationService.getDocumentation(unsupported);
        
        assertThat(docs).isNotEmpty();
        // Verify English content is returned as fallback
    }
}
```

**Documentation Endpoint Integration Test**:
```java
@QuarkusTest
class DocumentationEndpointTest {

    @Test
    void shouldReturnLocalizedDocumentationWithAcceptLanguageHeader() {
        given()
            .header("Accept-Language", "es")
            .header("Authorization", "Bearer " + getAdminToken())
        .when()
            .get("/api/v1/users/api-info")
        .then()
            .statusCode(200)
            .body("operations.createUser.summary", notNullValue());
    }

    @Test
    void shouldReturnEnglishDocumentationByDefault() {
        given()
            .header("Authorization", "Bearer " + getAdminToken())
        .when()
            .get("/api/v1/users/api-info")
        .then()
            .statusCode(200)
            .body("operations.createUser.summary", containsString("Create"));
    }
}
```

# Additional Recommendations:

- Cache complete documentation structures per locale with weak references to prevent memory leaks
- Establish automated workflows for detecting new API documentation keys during development
- Create validation scripts to ensure translation completeness before deployment
- Monitor documentation endpoint usage patterns by locale for capacity planning
- Integrate with translation management systems (TMS) for professional workflows

**References:**
- OpenAPI Specification v3.0.3: https://spec.openapis.org/oas/v3.0.3
- Quarkus OpenAPI & Swagger UI Guide: https://quarkus.io/guides/openapi-swaggerui
- SmallRye OpenAPI: https://github.com/smallrye/smallrye-open-api
- GraalVM Native Image Compatibility: https://www.graalvm.org/latest/reference-manual/native-image/metadata/Compatibility/
- Java 21 `ResourceBundle` Javadoc: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ResourceBundle.html

# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| OpenAPI Extension | `grep -r "smallrye-openapi\|quarkus-openapi" pom.xml */pom.xml` | SmallRye OpenAPI present |
| OpenAPI Annotations | `grep -r "@Operation\|@Tag" */src/main/java` | @Operation, @Tag annotations used |
| Swagger UI Config | `grep -r "quarkus.swagger-ui" */src/main/resources/application*.properties` | Swagger UI configured |
| OpenAPI JSON | `curl -sf localhost:8080/q/openapi` | OpenAPI spec generated |
| API Descriptions | `grep -r 'description\s*=' */src/main/java/**/api` | Descriptions in annotations |
| API Message Keys | `grep -r "MessageKeys.Api" */src/main/java` | Type-safe API doc keys used |
| API Translations | `grep -r "^api\." */src/main/resources/*messages*.properties` | API keys in all locale bundles |
| ApiDocumentationService | `grep -r "ApiDocumentationService" */src/main/java` | Runtime localization service exists |
| Documentation Tests | `./mvnw test -Dtest=*ApiDoc*,*DocumentationEndpoint*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating API Documentation & Localization..."
FAIL=0

# Detect module paths
if [ -f docker-compose.yml ] || [ -f docker-compose.yaml ] || [ -f compose.yml ]; then
  SRC_PATHS=$(find . -path "*/src/main/java" -not -path "*/node_modules/*" 2>/dev/null | head -20)
  RES_PATHS=$(find . -path "*/src/main/resources" -not -path "*/node_modules/*" 2>/dev/null | head -20)
else
  SRC_PATHS="./src/main/java"
  RES_PATHS="./src/main/resources"
fi

# Check for OpenAPI extension
if grep -rq "smallrye-openapi\|quarkus-openapi" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ SmallRye OpenAPI extension present"
else
  echo "❌ OpenAPI extension not found"
  FAIL=1
fi

# Check for OpenAPI annotations
ANNOTATIONS=("@Operation" "@Tag" "@APIResponse" "@Schema")
for anno in "${ANNOTATIONS[@]}"; do
  FOUND=0
  for sp in $SRC_PATHS; do
    if grep -rq "$anno" "$sp" 2>/dev/null; then FOUND=1; break; fi
  done
  if [ "$FOUND" -eq 1 ]; then
    echo "✅ $anno annotation used"
  else
    echo "❌ $anno annotation not found"
    FAIL=1
  fi
done

# Check for API descriptions in annotations
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq 'description\s*=' "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ API descriptions found in annotations"
else
  echo "❌ API descriptions missing"
  FAIL=1
fi

# Check for MessageKeys.Api usage
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "MessageKeys.Api\|MessageKeys\.Api" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ MessageKeys.Api constants found"
else
  echo "❌ No MessageKeys.Api constants found"
  FAIL=1
fi

# Check for ApiDocumentationService
FOUND=0
for sp in $SRC_PATHS; do
  if grep -rq "ApiDocumentationService" "$sp" 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ ApiDocumentationService found"
else
  echo "❌ ApiDocumentationService not found"
  FAIL=1
fi

# Check for API doc translations in resource bundles
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "^api\." "$rp"/*messages*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ API documentation keys in message bundles"
else
  echo "❌ No API documentation keys in message bundles"
  FAIL=1
fi

# Check Swagger UI config
FOUND=0
for rp in $RES_PATHS; do
  if grep -rq "quarkus.swagger-ui" "$rp"/application*.properties 2>/dev/null; then FOUND=1; break; fi
done
if [ "$FOUND" -eq 1 ]; then
  echo "✅ Swagger UI configured"
else
  echo "❌ Swagger UI not explicitly configured"
  FAIL=1
fi

# Check for @RegisterForReflection on documentation DTOs
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

# Run documentation tests
if ./mvnw test -Dtest="*ApiDoc*,*DocumentationEndpoint*" -q 2>/dev/null; then
  echo "✅ API documentation tests pass"
else
  echo "❌ API documentation tests failed or not found"
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
  echo "❌ API Documentation & Localization validation failed"
  exit 1
fi
echo "✅ All API Documentation & Localization criteria met"
```
