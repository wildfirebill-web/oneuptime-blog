# How to Version Control ClickHouse Schemas

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Schema Version Control, Git, DevOps, Database

Description: Learn how to version control ClickHouse schemas using Git, migration files, and schema dump tools to keep your database in sync with your codebase.

---

Version controlling ClickHouse schemas ensures that database structure changes are auditable, reproducible, and deployable alongside application code. The approach mirrors standard practices for relational databases adapted to ClickHouse's DDL.

## What to Version Control

- Table creation scripts (`CREATE TABLE` statements)
- Migration files (versioned up/down scripts)
- Dictionary definitions
- Materialized view definitions
- User and role definitions (non-sensitive parts)

## Directory Structure

```text
repo/
  database/
    schema/
      tables/
        events.sql
        sessions.sql
        materialized_views/
          mv_hourly_stats.sql
      migrations/
        0001_create_events.up.sql
        0001_create_events.down.sql
        0002_add_session_column.up.sql
        0002_add_session_column.down.sql
    README.md
```

## Dumping Current Schema

Extract existing table definitions:

```bash
# Dump a single table
clickhouse-client --query "SHOW CREATE TABLE analytics.events" \
  > database/schema/tables/events.sql

# Dump all tables in a database
for table in $(clickhouse-client --query "SHOW TABLES FROM analytics"); do
  clickhouse-client --query "SHOW CREATE TABLE analytics.$table" \
    > database/schema/tables/$table.sql
done
```

## Schema Drift Detection

Compare current schema against versioned files:

```bash
#!/usr/bin/env bash
# check-drift.sh
for sql_file in database/schema/tables/*.sql; do
  table=$(basename "$sql_file" .sql)
  current=$(clickhouse-client --query "SHOW CREATE TABLE analytics.$table")
  stored=$(cat "$sql_file")
  if [ "$current" != "$stored" ]; then
    echo "DRIFT DETECTED: $table"
    diff <(echo "$stored") <(echo "$current")
  fi
done
```

## Commit Conventions

Use descriptive commit messages for schema changes:

```text
feat(db): add country column to events table
fix(db): correct index granularity on sessions
refactor(db): rename user_uid to user_uuid
```

## Tagging Releases

Tag the repo when a production migration is applied:

```bash
git tag -a "db-v1.5.0" -m "Add country column and session index"
git push origin "db-v1.5.0"
```

## Using Git Hooks for Schema Validation

```bash
# .git/hooks/pre-commit
# Validate SQL files are parseable
for f in $(git diff --cached --name-only | grep '\.sql$'); do
  clickhouse-format < "$f" > /dev/null || {
    echo "Invalid SQL in $f"
    exit 1
  }
done
```

## Automating Schema Dumps in CI

Add a CI step to keep schema files in sync:

```yaml
# .github/workflows/dump-schema.yml
- name: Dump ClickHouse schema
  run: |
    for table in $(clickhouse-client --query "SHOW TABLES FROM analytics"); do
      clickhouse-client --query "SHOW CREATE TABLE analytics.$table" \
        > database/schema/tables/$table.sql
    done
    git diff --exit-code database/schema/ || {
      echo "Schema files are out of sync - run schema dump locally"
      exit 1
    }
```

## Summary

Version controlling ClickHouse schemas with Git, structured directories, and schema dump automation ensures that every environment can be reproduced from source, schema drift is caught early, and changes are reviewed like any other code change.
