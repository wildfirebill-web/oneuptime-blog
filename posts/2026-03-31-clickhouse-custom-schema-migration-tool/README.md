# How to Build a Custom Schema Migration Tool for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Schema Migration, Custom Tool, Python, DevOps

Description: Learn how to build a lightweight custom schema migration tool for ClickHouse in Python, tracking applied migrations and supporting rollbacks.

---

When off-the-shelf migration tools don't fit your workflow, building a custom solution for ClickHouse is straightforward. ClickHouse's idempotent DDL commands and simple HTTP interface make it an ideal target.

## Design Principles

- Store migration state in ClickHouse itself (a `schema_migrations` table)
- Migrations are SQL files named with a version prefix
- Applied in order, with rollback scripts for each migration
- Idempotent - safe to re-run

## Migration Tracking Table

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
    version     String,
    description String,
    applied_at  DateTime DEFAULT now(),
    checksum    String
)
ENGINE = ReplacingMergeTree(applied_at)
ORDER BY version;
```

## Directory Structure

```text
migrations/
  0001_create_events_table.up.sql
  0001_create_events_table.down.sql
  0002_add_session_id.up.sql
  0002_add_session_id.down.sql
```

## Migration Runner in Python

```python
import os
import hashlib
from clickhouse_driver import Client

client = Client('localhost', database='analytics')

MIGRATIONS_DIR = './migrations'

def get_applied_versions():
    rows = client.execute(
        "SELECT version FROM schema_migrations FINAL ORDER BY version"
    )
    return {row[0] for row in rows}

def run_sql_file(path):
    with open(path) as f:
        sql = f.read()
    for statement in sql.split(';'):
        stmt = statement.strip()
        if stmt:
            client.execute(stmt)

def compute_checksum(path):
    with open(path, 'rb') as f:
        return hashlib.md5(f.read()).hexdigest()

def migrate_up():
    applied = get_applied_versions()
    files = sorted(f for f in os.listdir(MIGRATIONS_DIR) if f.endswith('.up.sql'))

    for filename in files:
        version = filename.split('_')[0]
        if version in applied:
            print(f"  skip: {filename}")
            continue

        filepath = os.path.join(MIGRATIONS_DIR, filename)
        print(f"  apply: {filename}")
        run_sql_file(filepath)

        description = '_'.join(filename.replace('.up.sql', '').split('_')[1:])
        checksum = compute_checksum(filepath)
        client.execute(
            "INSERT INTO schema_migrations (version, description, checksum) VALUES",
            [{'version': version, 'description': description, 'checksum': checksum}]
        )

def migrate_down(steps=1):
    applied = sorted(get_applied_versions(), reverse=True)
    for version in applied[:steps]:
        files = [f for f in os.listdir(MIGRATIONS_DIR)
                 if f.startswith(version) and f.endswith('.down.sql')]
        if files:
            filepath = os.path.join(MIGRATIONS_DIR, files[0])
            print(f"  rollback: {files[0]}")
            run_sql_file(filepath)
            client.execute(
                "ALTER TABLE schema_migrations DELETE WHERE version = %(v)s",
                {'v': version}
            )

if __name__ == '__main__':
    import sys
    if len(sys.argv) > 1 and sys.argv[1] == 'down':
        migrate_down(int(sys.argv[2]) if len(sys.argv) > 2 else 1)
    else:
        migrate_up()
```

## Running Migrations

```bash
# Apply all pending migrations
python migrate.py up

# Roll back one migration
python migrate.py down 1
```

## Summary

A custom ClickHouse migration tool can be built in under 100 lines of Python, using ClickHouse itself to track state. This approach gives you full control over migration logic, retry behavior, and toolchain integration.
