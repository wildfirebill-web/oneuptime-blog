# How to Use Flyway for MySQL Database Migrations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Flyway, Migration, Database

Description: Learn how to set up Flyway for MySQL schema migrations, write versioned SQL scripts, and integrate migrations into a CI/CD pipeline.

---

Flyway is a Java-based database migration tool that applies versioned SQL scripts to a database in a controlled, repeatable order. It tracks applied migrations in a `flyway_schema_history` table.

## Installation

```bash
# macOS with Homebrew
brew install flyway

# Or download the community edition
curl -L https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.8.1/flyway-commandline-10.8.1-linux-x64.tar.gz | tar xvz
```

## Configuration

Create `flyway.conf` in your project root:

```text
flyway.url=jdbc:mysql://localhost:3306/myapp
flyway.user=appuser
flyway.password=secret
flyway.locations=filesystem:./db/migrations
flyway.baselineOnMigrate=true
flyway.validateOnMigrate=true
```

## Migration File Naming

Flyway enforces a strict naming convention:

```text
V{version}__{description}.sql
```

Examples:
- `V1__create_users_table.sql`
- `V2__add_email_index.sql`
- `V3__create_orders_schema.sql`

Version numbers must be unique and increase monotonically.

## Writing Migration Scripts

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email      VARCHAR(255) NOT NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```sql
-- V2__add_name_to_users.sql
ALTER TABLE users
    ADD COLUMN first_name VARCHAR(100) NOT NULL DEFAULT '' AFTER email,
    ADD COLUMN last_name  VARCHAR(100) NOT NULL DEFAULT '' AFTER first_name;
```

## Running Migrations

```bash
# Apply all pending migrations
flyway migrate

# Check current state without applying
flyway info

# Validate applied checksums match files
flyway validate

# Undo last migration (requires Flyway Teams)
flyway undo
```

## Integrating With Maven (Spring Boot)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

Spring Boot auto-runs Flyway on startup when the dependency is on the classpath. Configure via `application.properties`:

```text
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
```

## Repeatable Migrations

For views and stored procedures that should be re-applied when the file changes, use `R__` prefix:

```sql
-- R__create_active_users_view.sql
CREATE OR REPLACE VIEW active_users AS
SELECT id, email, first_name, last_name
FROM   users
WHERE  deleted_at IS NULL;
```

## CI/CD Integration

```bash
# In your CI pipeline
flyway -url="jdbc:mysql://$DB_HOST:3306/$DB_NAME" \
       -user="$DB_USER" \
       -password="$DB_PASS" \
       migrate
```

## Summary

Flyway manages MySQL schema evolution through versioned SQL files stored in version control. Name files with `V{version}__{description}.sql` convention. Run `flyway migrate` to apply pending migrations. Use `R__` prefix for repeatable migrations like views. Integrate with CI/CD by running `flyway migrate` as a step before deploying your application.
