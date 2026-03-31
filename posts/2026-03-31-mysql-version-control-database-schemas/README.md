# How to Version Control MySQL Database Schemas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Version Control, Schema Migration, Git

Description: Learn how to version control MySQL schemas with Git, migration tools, and CI/CD integration for reproducible database deployments.

---

Treating database schemas with the same discipline as application code - commit history, pull requests, code review, and CI/CD automation - prevents schema drift and makes rollbacks predictable.

## The Core Principle: Schema as Code

Every schema change must be expressed as a versioned migration file committed to Git. No ad-hoc changes directly on production servers.

## Directory Structure

```text
project/
  db/
    migrations/
      V1__initial_schema.sql
      V2__add_users_table.sql
      V3__add_orders_table.sql
      V4__add_index_orders_status.sql
    seeds/
      01_roles.sql
      02_default_config.sql
    schema.sql          <- current full schema snapshot
```

## Using Flyway With Git

```bash
# Each migration is a commit
git add db/migrations/V5__add_audit_log_table.sql
git commit -m "feat(db): add audit_log table for compliance tracking"
```

```bash
# Apply on deploy
flyway migrate
```

## Using skeema for Declarative Schema

skeema stores the schema as one `.sql` file per table, letting you diff changes like regular code:

```sql
-- db/tables/users.sql (current desired state)
CREATE TABLE users (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email      VARCHAR(255) NOT NULL,
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_email (email)
) ENGINE=InnoDB;
```

When you add a column, you edit the file:

```sql
-- Add `name` column directly in the file
CREATE TABLE users (
    id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
    email      VARCHAR(255) NOT NULL,
    name       VARCHAR(150) NOT NULL DEFAULT '',
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_email (email)
) ENGINE=InnoDB;
```

skeema generates and applies the ALTER automatically.

## CI/CD Integration

```yaml
# .github/workflows/db-migrate.yml
name: Database Migration

on:
  push:
    branches: [main]
    paths:
      - 'db/migrations/**'

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Flyway migrations
        env:
          DB_URL: ${{ secrets.DB_URL }}
          DB_USER: ${{ secrets.DB_USER }}
          DB_PASS: ${{ secrets.DB_PASS }}
        run: |
          flyway -url="$DB_URL" -user="$DB_USER" -password="$DB_PASS" migrate
```

## Keeping Schema.sql in Sync

After each successful migration run, regenerate the full schema snapshot:

```bash
mysqldump --no-data --skip-comments \
          -u root myapp > db/schema.sql
git add db/schema.sql
git commit -m "chore(db): regenerate schema snapshot after V5 migration"
```

## Pull Request Checklist for Schema Changes

```text
- [ ] Migration file follows naming convention
- [ ] Rollback SQL is written and tested
- [ ] No unsafe ALTER TABLE on tables with > 1M rows (use pt-osc/gh-ost)
- [ ] New indexes added with ALGORITHM=INPLACE
- [ ] Migration tested on staging database
- [ ] schema.sql updated
```

## Summary

Version control MySQL schemas by storing every change as a migration file committed to Git. Use Flyway or Liquibase for versioned migrations and skeema for declarative schema management. Automate migrations in CI/CD pipelines triggered by changes to the migrations directory. Keep a `schema.sql` snapshot updated after every migration for quick onboarding and diff reviews.
