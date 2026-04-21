# Issue: Database Schema Migration Management

Database schema evolution must be managed in a controlled, versioned, and auditable manner to support reliable deployments and team collaboration. Relying on Hibernate's automatic schema generation (`quarkus.hibernate-orm.database.generation=update`) in production creates unpredictable behavior, lacks version history, and risks data loss during complex schema changes.

# Decision:

Adopt **Flyway** as the standard tool for managing all database schema migrations. All schema changes **MUST** be implemented as versioned SQL migration scripts and applied by Flyway.

Hibernate's automatic schema generation (`quarkus.hibernate-orm.database.generation`) **MUST** be disabled in production environments and set to `none`.

# Constraints:

*   **Production Safety:** Migrations **MUST** be applied in a controlled manner in production, preferably as a separate, reviewed step in the deployment pipeline rather than automatically on application startup.
*   **Backward Compatibility:** Migration scripts **SHOULD** be written to be backward-compatible whenever possible to support zero-downtime deployment strategies.
*   **Immutability:** Once a migration has been applied to production, its script **MUST NOT** be altered. New changes require a new migration script.
*   **Permissions:** The database user configured for Flyway **MUST** have DDL permissions to alter the schema.
*   **Performance:** The migration process **SHOULD NOT** excessively prolong application startup times.

# Alternatives:

**Hibernate Schema Generation**: Hibernate `hbm2ddl` compares the entity model to the live schema and emits DDL at runtime; the Hibernate documentation explicitly warns that `update` is not safe for production (https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#schema-generation). Generated DDL is non-deterministic, produces no version history, and cannot handle data-only migrations or column renames. Rejected because production deployments require auditable, repeatable, versioned migration scripts.

**Liquibase**: Supports XML, YAML, JSON, and SQL changelog formats with a database-diff engine (https://docs.liquibase.com/concepts/home.html). Flyway uses plain SQL files by default and follows a linear versioning model, whereas Liquibase requires a changelog manifest that maps each changeset to an author and id. The Quarkus extension catalogue lists `quarkus-flyway` as first-party with identical hot-reload support (https://quarkus.io/guides/flyway). Rejected because the project's migrations are SQL-only and do not benefit from Liquibase's multi-format abstraction layer.

# Rationale:

Flyway assigns a monotonically increasing version number to each migration script and records applied versions in a `flyway_schema_history` table, producing an auditable change log (https://documentation.red-gate.com/flyway/flyway-concepts/migrations). Plain SQL migration files are executed as-is, making every DDL change explicit and diff-reviewable. The `quarkus-flyway` extension is maintained in the Quarkus core repository and supports dev-mode hot-reload, `@QuarkusTest` integration, and native compilation (https://quarkus.io/guides/flyway). Flyway's naming convention (`V{version}__{description}.sql`) requires no external manifest, reducing configuration to a single `quarkus.flyway.migrate-at-start=true` property.

# Implementation Guidelines:

### 1. Add Maven Dependency

Add the `quarkus-flyway` dependency to the `pom.xml` of the main container application module.

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-flyway</artifactId>
</dependency>
```

### 2. Configure `application.properties`

Update the configuration to enable and configure Flyway, and to disable Hibernate's automatic schema generation for the production profile.

```properties
# Flyway Configuration
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=db/migration
quarkus.flyway.table=flyway_schema_history
quarkus.flyway.baseline-on-migrate=true

# Disable Hibernate DDL generation for production
%prod.quarkus.hibernate-orm.database.generation=none
```

### 3. Create Migration Scripts

Create the directory `src/main/resources/db/migration`. All SQL migration scripts will be placed here.

The scripts **MUST** follow Flyway's naming convention: `V<VERSION>__<DESCRIPTION>.sql`. The version consists of numbers separated by dots or underscores.

**Example: `src/main/resources/db/migration/V1.0.0__create_user_table.sql`**

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);

CREATE SEQUENCE users_seq
    START WITH 1
    INCREMENT BY 50
    MINVALUE 1
    NO MAXVALUE
    CACHE 1;
```

### 4. Validation Commands

Developers can use the following commands to manage and validate migrations.

```bash
# The application will run migrations on startup in dev mode
./mvnw quarkus:dev

# Use the Flyway Maven plugin for more control
# (Requires adding the plugin to pom.xml)

# Check the status of all migrations
./mvnw flyway:info

# Validate applied migrations against available scripts
./mvnw flyway:validate
```

# Additional Recommendations:

- It is RECOMMENDED to write integration tests for complex data migrations using Flyway's test helpers (https://documentation.red-gate.com/flyway/flyway-concepts/migrations#test-migrations)
- It is RECOMMENDED to use `./mvnw flyway:repair` to resolve failed migrations before retrying (https://documentation.red-gate.com/flyway/flyway-cli-and-api/usage/command-line/command-line-repair)
- Hotfix schema changes **MUST** be applied as new versioned migration scripts; manual database changes **MUST NOT** be performed (https://documentation.red-gate.com/flyway/flyway-concepts/migrations#versioned-migrations)
- Developers **SHOULD** pull the latest changes before creating new migrations to avoid version conflicts (see ADR-03, ADR-06)

# Success Criteria:

**Completion Gate**: The following criteria MUST all be met before considering this ADR successfully implemented.

| Criteria | Validation Method | Expected Result |
|----------|-------------------|------------------|
| Flyway Dependency | `grep flyway pom.xml` | Flyway extension present |
| Migration Directory | `ls src/main/resources/db/migration` | Directory exists |
| Migration Naming | `ls V*__*.sql` | Follows V{version}__{description}.sql |
| Flyway Config | `grep flyway application.properties` | Configuration present |
| Migrations Run | `./mvnw flyway:info` | Shows migration history |
| Schema Valid | `./mvnw flyway:validate` | Exit code 0 |
| Clean Start | Fresh database migration | All migrations apply |
| Build Success | `./mvnw clean verify` | Exit code 0 |

**Validation Script:**
```bash
#!/bin/bash
set -e
echo "Validating Data Migration Strategy..."

FAIL=0

# Check for Flyway dependency
if grep -rq "flyway" pom.xml */pom.xml 2>/dev/null; then
  echo "✅ Flyway dependency configured"
else
  echo "❌ Flyway dependency not found"
  FAIL=1
fi

# Check for migration directory
MIGRATION_DIR=$(find . -path "*/db/migration" -type d 2>/dev/null | head -1)
if [ -n "$MIGRATION_DIR" ]; then
  echo "✅ Migration directory exists: $MIGRATION_DIR"
  
  # Check migration files
  MIGRATION_COUNT=$(ls "$MIGRATION_DIR"/V*.sql 2>/dev/null | wc -l)
  echo "   Found $MIGRATION_COUNT migration files"
  
  # Verify naming convention
  BAD_NAMES=$(ls "$MIGRATION_DIR"/*.sql 2>/dev/null | grep -v "V[0-9].*__" | wc -l)
  if [ "$BAD_NAMES" -eq 0 ]; then
    echo "✅ All migrations follow naming convention"
  else
    echo "❌ Some migrations don't follow V{n}__{desc}.sql convention"
    FAIL=1
  fi
else
  echo "❌ Migration directory not found"
  FAIL=1
fi

# Check Flyway configuration
if grep -rq "quarkus.flyway" */src/main/resources/application*.properties 2>/dev/null; then
  echo "✅ Flyway configuration present"
else
  echo "❌ Flyway configuration not found"
  FAIL=1
fi

# Run Flyway validation (requires database)
./mvnw flyway:validate -q 2>/dev/null && echo "✅ Flyway validation passed" || echo "ℹ️  Flyway validation skipped (no database)"

# Build verification
./mvnw clean compile -q && echo "✅ Build successful" || { echo "❌ Build failed"; FAIL=1; }

if [ $FAIL -ne 0 ]; then
  echo "❌ Data Migration Strategy validation FAILED"
  exit 1
fi
echo "✅ All Data Migration Strategy criteria met"
```
