# Issue: Security Implementation using OWASP Top 10 Guidelines

Quarkus reactive microservices require comprehensive OWASP Top 10 security implementation with performance constraints and hexagonal architecture compliance.

# Decision:

Implement comprehensive OWASP Top 10 security using Quarkus Security with reactive programming support, enforcing security at adapter boundaries while maintaining domain independence.

**Required Components**:
- JWT authentication with RS256 (RSA-2048 keys)
- Role-based access control with least privilege
- Comprehensive input validation and sanitization
- AES-256-GCM encryption for sensitive data
- Security audit logging with OpenTelemetry
- Distributed rate limiting and brute force protection
- Container security hardening
- Automated dependency vulnerability scanning (blocks CVSS 7+)
- SSRF prevention with URL validation
- Security headers and TLS 1.3+

# Constraints:

- Security enforcement **MUST NOT** introduce blocking operations in reactive streams
- All security components **MUST** be compatible with GraalVM native compilation
- Security configuration **MUST** be resolved at build time where possible
- Domain layer **MUST** remain unaware of security infrastructure (see ADR-10)
- Security enforcement **MUST** occur at adapter boundaries only
- Environment-specific security configuration **MUST** be externalized via environment variables (see ADR-15)
- Key rotation **MUST** be supported without application restarts

# Alternatives:

**Spring Security with Reactive Support**: Spring Security provides `ReactiveSecurityContextHolder` for WebFlux applications (https://docs.spring.io/spring-security/reference/reactive/index.html). However, Spring Security requires the Spring Framework runtime, which is incompatible with the Quarkus build-time CDI model and ArC compile-time dependency injection (https://quarkus.io/guides/cdi-reference#limitations). Quarkus applications using Spring Security cannot produce GraalVM native images via the standard `quarkus-maven-plugin` build. Rejected because the project uses Quarkus with native compilation (see ADR-08).

**Custom Security Implementation**: Building authentication, authorization, and cryptographic services from scratch using raw JDK security APIs (`java.security`, `javax.crypto`). OWASP explicitly recommends using established frameworks over custom implementations to avoid introducing vulnerabilities through incorrect cryptographic usage or authorization logic flaws (https://owasp.org/Top10/A02_2021-Cryptographic_Failures/). Rejected because Quarkus Security and SmallRye JWT provide tested, specification-compliant implementations that reduce the attack surface.

# Rationale:

Quarkus Security resolves `@RolesAllowed`, `@Authenticated`, and `@PermitAll` annotations at build time via ArC, eliminating runtime reflection for authorization checks (https://quarkus.io/guides/security-overview). SmallRye JWT implements the MicroProfile JWT RBAC 2.1 specification for RS256/RS384/RS512 token verification without external libraries (https://quarkus.io/guides/security-jwt). The `quarkus.http.header.*` configuration injects OWASP-recommended response headers (X-Content-Type-Options, X-Frame-Options, HSTS) at the Vert.x HTTP layer before application code executes (https://quarkus.io/guides/http-reference#additional-http-headers). OWASP Dependency-Check Maven plugin scans transitive dependencies against the NVD and fails the build on CVSS ≥ 7.0 (https://jeremylong.github.io/DependencyCheck/dependency-check-maven/). These components map to OWASP Top 10 categories A01 (Broken Access Control), A02 (Cryptographic Failures), A03 (Injection), A06 (Vulnerable Components), and A07 (Authentication Failures).

# Implementation Guidelines:

## 1. JWT Authentication Infrastructure

**RSA Key Generation**:
```bash
# Generate RSA-2048 private key
openssl genpkey -algorithm RSA -out private.pem -pkcs8 -outform PEM -pkeyopt rsa_keygen_bits:2048
openssl rsa -in private.pem -pubout -out public.pem

# Place keys
mkdir -p src/main/resources/META-INF/resources/
cp private.pem src/main/resources/META-INF/resources/privateKey.pem
cp public.pem src/main/resources/META-INF/resources/publicKey.pem

# Verify
openssl rsa -in src/main/resources/META-INF/resources/privateKey.pem -check -noout
```

**Application Configuration**:
```properties
# JWT Security
mp.jwt.verify.publickey.location=META-INF/resources/publicKey.pem
mp.jwt.verify.issuer=https://copilot-quarkus-demo
smallrye.jwt.sign.key.location=META-INF/resources/privateKey.pem

# Security Headers
quarkus.http.header."X-Content-Type-Options".value=nosniff
quarkus.http.header."X-Frame-Options".value=DENY
quarkus.http.header."X-XSS-Protection".value=1; mode=block
quarkus.http.header."Strict-Transport-Security".value=max-age=31536000; includeSubDomains
quarkus.http.insecure-requests=disabled

# OpenAPI Security
quarkus.smallrye-openapi.path=/openapi
quarkus.swagger-ui.always-include=true
quarkus.swagger-ui.path=/swagger-ui
```

**Naming Conventions**:
- Controllers: `[Domain]SecurityController`
- Services: `[Function]SecurityService` 
- DTOs: `[Action]SecurityRequest/Response`
- Exceptions: `[Type]SecurityException`
- Validators: `[Domain]SecurityValidator`
- Audit events: `[Action]SecurityAuditEvent`

## 2. Authentication Controller Implementation

**JWT Token Service**:
```java
@ApplicationScoped
public class JwtTokenService {
    private static final Logger logger = LoggerFactory.getLogger(JwtTokenService.class);
    
    public String generateAdminToken() {
        return Jwt.issuer("https://copilot-quarkus-demo")
                .upn("admin")
                .groups(Set.of("ADMIN", "USER"))
                .expiresIn(Duration.ofHours(1))
                .sign();
    }
    
    public String generateUserToken() {
        return Jwt.issuer("https://copilot-quarkus-demo")
                .upn("user") 
                .groups(Set.of("USER"))
                .expiresIn(Duration.ofHours(1))
                .sign();
    }
    
    public String generateCustomToken(String username, Set<String> roles, Duration expiration) {
        logger.info("Generating token for user: {} with roles: {}", username, roles);
        return Jwt.issuer("https://copilot-quarkus-demo")
                .upn(username)
                .groups(roles)
                .expiresIn(expiration)
                .sign();
    }
}
```

**Authentication Controller**:
```java
@Path("/api/auth")
@Tag(name = "Authentication", description = "JWT token generation for development/testing")
public class AuthController {
    @Inject
    JwtTokenService jwtTokenService;
    
    @GET
    @Path("/token/admin")
    @Operation(summary = "Generate admin JWT token")
    public Response generateAdminToken() {
        String token = jwtTokenService.generateAdminToken();
        return Response.ok(Map.of(
            "access_token", token,
            "token_type", "Bearer", 
            "expires_in", 3600,
            "username", "admin",
            "roles", Set.of("ADMIN", "USER")
        )).build();
    }
    
    @GET
    @Path("/token/user") 
    @Operation(summary = "Generate user JWT token")
    public Response generateUserToken() {
        String token = jwtTokenService.generateUserToken();
        return Response.ok(Map.of(
            "access_token", token,
            "token_type", "Bearer",
            "expires_in", 3600, 
            "username", "user",
            "roles", Set.of("USER")
        )).build();
    }
}
```

## 3. Access Control Implementation

**Secured REST Controller**:
```java
@Path("/api/v1/users")
@Tag(name = "Users", description = "User management operations")
@SecurityRequirement(name = "jwt")
public class UserController {
    @Inject
    UserService userService;
    
    @Inject
    SecurityAuditLogger securityAuditLogger;
    
    @GET
    @RolesAllowed({"ADMIN"})
    public Uni<List<UserResponse>> getAllUsers(@Context SecurityContext securityContext) {
        String currentUser = securityContext.getUserPrincipal().getName();
        securityAuditLogger.logDataAccess("user", "all", currentUser, "READ_ALL");
        
        return userService.getAllUsers()
            .onItem().transform(users -> users.stream()
                .map(UserResponse::from)
                .collect(Collectors.toList()));
    }
    
    @GET
    @Path("/{id}")
    @RolesAllowed({"ADMIN", "USER"})
    public Uni<UserResponse> getUserById(@PathParam("id") Long id, 
                                        @Context SecurityContext securityContext) {
        String currentUser = securityContext.getUserPrincipal().getName();
        boolean isAdmin = securityContext.isUserInRole("ADMIN");
        
        return checkUserAccess(id, currentUser, isAdmin)
            .onItem().transformToUni(accessAllowed -> {
                if (!accessAllowed) {
                    securityAuditLogger.logAuthorizationFailure(
                        "user", String.valueOf(id), currentUser, "READ", "INSUFFICIENT_PRIVILEGES");
                    return Uni.createFrom().failure(
                        new WebApplicationException(Response.status(403).build()));
                }
                
                securityAuditLogger.logDataAccess("user", String.valueOf(id), currentUser, "READ");
                return userService.getUserById(id)
                    .onItem().transform(UserResponse::from);
            });
    }
    
    private Uni<Boolean> checkUserAccess(Long userId, String currentUser, boolean isAdmin) {
        if (isAdmin) return Uni.createFrom().item(true);
        return userService.getUserByUsername(currentUser)
            .onItem().transform(user -> user != null && user.getId().equals(userId));
    }
}
```

## 4. Input Validation and Sanitization

**DTO Validation**:
```java
@Schema(description = "Request to create a new user")
public class CreateUserRequest {
    @Email(message = "validation.email.invalid")
    @NotBlank(message = "validation.email.required")
    @Pattern(regexp = "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", 
             message = "validation.email.format")
    @Schema(description = "User's email address", example = "user@example.com")
    private String email;
    
    @NotBlank(message = "validation.firstName.required")
    @Size(min = 1, max = 100, message = "validation.firstName.size")
    @Pattern(regexp = "^[a-zA-ZÀ-ÿ\\s'-]+$", message = "validation.firstName.format")
    @Schema(description = "User's first name", example = "John")
    private String firstName;
    
    @NotBlank(message = "validation.lastName.required") 
    @Size(min = 1, max = 100, message = "validation.lastName.size")
    @Pattern(regexp = "^[a-zA-ZÀ-ÿ\\s'-]+$", message = "validation.lastName.format")
    @Schema(description = "User's last name", example = "Doe")
    private String lastName;
}
```

**Input Validation Service**:
```java
@ApplicationScoped
public class InputValidationService {
    private static final Pattern SQL_INJECTION = Pattern.compile(
        "(?i)(union|select|insert|update|delete|drop|create|alter|exec|execute)", 
        Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);
    private static final Pattern XSS = Pattern.compile(
        "(?i)(<script|</script|javascript:|vbscript:|onload=|onerror=)", 
        Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);
    private static final Pattern COMMAND_INJECTION = Pattern.compile(
        "(?i)(;|&&|\\|\\||`|\\$\\(|\\$\\{|\\\\\\\\|\\.\\.|\\.\\.?/|cmd|powershell|bash|sh|rm|del)", 
        Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);
    
    public boolean isValidEmail(String email) {
        return email != null && 
               email.matches("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$") &&
               !containsMaliciousPatterns(email);
    }
    
    public boolean isValidName(String name) {
        return name != null &&
               name.matches("^[a-zA-ZÀ-ÿ\\s'-]+$") &&
               name.length() >= 1 && name.length() <= 100 &&
               !containsMaliciousPatterns(name);
    }
    
    public String sanitizeInput(String input) {
        if (input == null) return null;
        return input.replaceAll("[<>\"'&]", "")
                   .trim()
                   .substring(0, Math.min(input.length(), 1000));
    }
    
    private boolean containsMaliciousPatterns(String input) {
        return SQL_INJECTION.matcher(input).find() ||
               XSS.matcher(input).find() ||
               COMMAND_INJECTION.matcher(input).find();
    }
}
```

## 5. Cryptographic Services Implementation

**Data Encryption Service**:
```java
@ApplicationScoped
public class CryptographicService {
    @ConfigProperty(name = "app.encryption.algorithm", defaultValue = "AES/GCM/NoPadding")
    String encryptionAlgorithm;
    @ConfigProperty(name = "app.encryption.key-size", defaultValue = "256") 
    int keySize;
    
    private final SecureRandom secureRandom = new SecureRandom();
    private static final Logger logger = LoggerFactory.getLogger(CryptographicService.class);
    
    public String encryptSensitiveData(String plaintext, SecretKey secretKey) {
        try {
            Cipher cipher = Cipher.getInstance(encryptionAlgorithm);
            byte[] iv = new byte[12];
            secureRandom.nextBytes(iv);
            
            GCMParameterSpec gcmSpec = new GCMParameterSpec(128, iv);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey, gcmSpec);
            
            byte[] encryptedData = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
            
            byte[] result = new byte[iv.length + encryptedData.length];
            System.arraycopy(iv, 0, result, 0, iv.length);
            System.arraycopy(encryptedData, 0, result, iv.length, encryptedData.length);
            
            return Base64.getEncoder().encodeToString(result);
        } catch (Exception e) {
            logger.error("Encryption failed", e);
            throw new CryptographicException("Encryption operation failed", e);
        }
    }
    
    public String decryptSensitiveData(String encryptedData, SecretKey secretKey) {
        try {
            byte[] data = Base64.getDecoder().decode(encryptedData);
            
            byte[] iv = new byte[12];
            byte[] encrypted = new byte[data.length - 12];
            System.arraycopy(data, 0, iv, 0, 12);
            System.arraycopy(data, 12, encrypted, 0, encrypted.length);
            
            Cipher cipher = Cipher.getInstance(encryptionAlgorithm);
            GCMParameterSpec gcmSpec = new GCMParameterSpec(128, iv);
            cipher.init(Cipher.DECRYPT_MODE, secretKey, gcmSpec);
            
            byte[] decryptedData = cipher.doFinal(encrypted);
            return new String(decryptedData, StandardCharsets.UTF_8);
        } catch (Exception e) {
            logger.error("Decryption failed", e);
            throw new CryptographicException("Decryption operation failed", e);
        }
    }
    
    public String hashPassword(String password) {
        return BCrypt.hashpw(password, BCrypt.gensalt(12));
    }
    
    public boolean verifyPassword(String password, String hashedPassword) {
        return BCrypt.checkpw(password, hashedPassword);
    }
    
    public SecretKey generateAESKey() {
        try {
            KeyGenerator keyGen = KeyGenerator.getInstance("AES");
            keyGen.init(keySize);
            return keyGen.generateKey();
        } catch (Exception e) {
            throw new CryptographicException("Key generation failed", e);
        }
    }
}

@ApplicationScoped
public class CryptographicException extends InfrastructureException {
    public CryptographicException(String message, Throwable cause) {
        super("SEC-001", "Cryptographic", message, cause);
    }
}
```

## 6. Security Audit Logging

**Security Audit Logger with OpenTelemetry**:
```java
@ApplicationScoped
public class SecurityAuditLogger {
    private static final Logger logger = LoggerFactory.getLogger(SecurityAuditLogger.class);
    
    public void logDataAccess(String resourceType, String resourceId, String userId, String operation) {
        Map<String, Object> eventData = Map.of(
            "event_type", "DATA_ACCESS",
            "resource_type", resourceType,
            "resource_id", resourceId, 
            "user_id", userId,
            "operation", operation,
            "timestamp", Instant.now(),
            "trace_id", Span.current().getSpanContext().getTraceId()
        );
        
        logger.info("Security audit: {} accessed {} {} (operation: {})", 
                   userId, resourceType, resourceId, operation);
        
        Span.current().addEvent("security.data_access", Attributes.of(
            AttributeKey.stringKey("user.id"), userId,
            AttributeKey.stringKey("resource.type"), resourceType,
            AttributeKey.stringKey("operation"), operation
        ));
    }
    
    public void logAuthorizationFailure(String resource, String resourceId, String userId, 
                                      String operation, String reason) {
        logger.warn("Authorization failed: user {} attempted {} on {} {} (reason: {})", 
                   userId, operation, resource, resourceId, reason);
        
        Span.current().addEvent("security.authorization_failure", Attributes.of(
            AttributeKey.stringKey("user.id"), userId,
            AttributeKey.stringKey("resource"), resource,
            AttributeKey.stringKey("operation"), operation,
            AttributeKey.stringKey("failure.reason"), reason
        ));
    }
    
    public void logSecurityViolation(String violationType, String details, String userId, 
                                   String clientIp, SecurityLevel level) {
        logger.error("Security violation [{}]: {} - User: {}, IP: {}, Details: {}", 
                    level, violationType, userId, clientIp, details);
        
        Span.current().addEvent("security.violation", Attributes.of(
            AttributeKey.stringKey("violation.type"), violationType,
            AttributeKey.stringKey("user.id"), userId,
            AttributeKey.stringKey("client.ip"), clientIp,
            AttributeKey.stringKey("severity"), level.name(),
            AttributeKey.stringKey("details"), details
        ));
    }
    
    public enum SecurityLevel {
        LOW, MEDIUM, HIGH, CRITICAL
    }
}
```

## 7. OpenAPI Security Configuration

**Security Scheme Definition**:
```java
@OpenAPIDefinition(
    info = @Info(
        title = "Copilot Quarkus Demo API",
        version = "1.0.0-SNAPSHOT",
        description = "Secure microservices API with OWASP Top 10 compliance"
    ),
    servers = {
        @Server(url = "http://localhost:8080", description = "Development server"),
        @Server(url = "https://api.example.com", description = "Production server") 
    }
)
@SecurityScheme(
    securitySchemeName = "jwt",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer", 
    bearerFormat = "JWT",
    description = "JWT Bearer token authentication. " +
                 "Obtain tokens from /api/auth/token/admin or /api/auth/token/user endpoints."
)
public class OpenApiConfig extends Application {
}
```

## 8. Dependency Security Configuration

**Maven Security Plugin**:
```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>10.0.4</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFiles>
            <suppressionFile>dependency-suppression.xml</suppressionFile>
        </suppressionFiles>
        <formats>
            <format>HTML</format>
            <format>JSON</format>
        </formats>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

**Validation Commands**:
```bash
# Generate tokens and test authentication
curl -X GET "http://localhost:8080/api/auth/token/admin"
TOKEN=$(curl -s "http://localhost:8080/api/auth/token/admin" | jq -r '.access_token')
curl -H "Authorization: Bearer $TOKEN" "http://localhost:8080/api/v1/users"

# Security validation
./mvnw org.owasp:dependency-check-maven:check
curl "http://localhost:8080/openapi" | jq '.components.securitySchemes'

# Input validation testing
curl -X POST "http://localhost:8080/api/v1/users" \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"email":"test@example.com","firstName":"<script>alert(1)</script>","lastName":"Test"}'
```

# Additional Recommendations:

- It is RECOMMENDED to use Redis-based distributed rate limiting for concurrent request management via the `quarkus-redis-client` extension (https://quarkus.io/guides/redis)
- It is RECOMMENDED to implement JWT token caching with TTL using `quarkus-cache` to reduce cryptographic overhead (https://quarkus.io/guides/cache)
- Container images **SHOULD** use distroless base images with non-root user execution (https://quarkus.io/guides/container-image#container-image-options)
- It is RECOMMENDED to configure Grafana security dashboards for authentication rates and suspicious activity (https://grafana.com/docs/grafana/latest/dashboards/)
- It is RECOMMENDED to integrate OWASP ZAP for automated penetration testing in CI/CD pipelines (https://www.zaproxy.org/docs/docker/about/)


# Success Criteria:

> **Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| JWT Extension | `grep smallrye-jwt pom.xml` | SmallRye JWT present |
| OIDC Extension | `grep oidc pom.xml` | OIDC extension present |
| Security Annotations | Code inspection | @RolesAllowed, @Authenticated used |
| CORS Configured | `grep cors application.properties` | CORS settings present |
| HTTPS Enforced | Configuration check | TLS configured for prod |
| Security Headers | Response headers | X-Content-Type-Options, etc. |
| Rate Limiting | Configuration/code | Rate limiting implemented |
| Input Validation | Code inspection | Bean Validation on endpoints |
| Security Tests | `./mvnw test -Dtest=*Security*` | All tests pass |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Security Implementation (OWASP)..."

FAIL=0

# Check for security extensions
SECURITY_FOUND=0
SECURITY_DEPS=("smallrye-jwt" "security")
for dep in "${SECURITY_DEPS[@]}"; do
  if grep -rq "$dep" pom.xml */pom.xml 2>/dev/null; then
    echo "✅ Security dependency '$dep' found"
    SECURITY_FOUND=1
  fi
done
if [ "$SECURITY_FOUND" -eq 0 ]; then
  echo "❌ No security dependencies found (need smallrye-jwt or quarkus-security)"
  FAIL=1
fi

# Check for security annotations
if grep -rq "@RolesAllowed\|@Authenticated\|@PermitAll\|@DenyAll" */src/main/java 2>/dev/null; then
  echo "✅ Security annotations in use"
else
  echo "❌ No security annotations found"
  FAIL=1
fi

# Check CORS configuration
if grep -rq "quarkus.http.cors" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ CORS configured"
else
  echo "❌ CORS not explicitly configured"
  FAIL=1
fi

# Check for HTTPS/TLS configuration
if grep -rq "quarkus.http.ssl\|quarkus.http.insecure-requests" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ TLS/HTTPS configuration found"
else
  echo "❌ No TLS configuration detected"
  FAIL=1
fi

# Check for input validation
if grep -rq "@Valid\|@NotNull\|@NotBlank\|@Size" */src/main/java 2>/dev/null; then
  echo "✅ Input validation annotations found"
else
  echo "❌ No input validation annotations"
  FAIL=1
fi

# Check for security headers configuration
if grep -rq "quarkus.http.header\|Content-Security-Policy" */src/main/resources 2>/dev/null; then
  echo "✅ Security headers configured"
else
  echo "❌ Security headers not explicitly configured"
  FAIL=1
fi

# Run security tests
./mvnw test -Dtest="*Security*,*Auth*" -q 2>/dev/null && echo "✅ Security tests pass" || { echo "❌ Security tests failed or not found"; FAIL=1; }

# Build verification
./mvnw clean compile -q && echo "✅ Build successful" || { echo "❌ Build failed"; FAIL=1; }

if [ $FAIL -ne 0 ]; then
  echo "❌ Security Implementation (OWASP) validation FAILED"
  exit 1
fi
echo "✅ All Security Implementation (OWASP) criteria met"
```
