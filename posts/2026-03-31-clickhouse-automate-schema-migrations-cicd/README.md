# How to Automate Schema Migrations in CI/CD for ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CI/CD, Schema Migration, Automation, DevOps

Description: Learn how to automate ClickHouse schema migrations in CI/CD pipelines using GitHub Actions, with safe apply, validation, and rollback steps.

---

Automating ClickHouse schema migrations in CI/CD ensures consistent, auditable deployments across environments. The pipeline should validate, apply, and verify migrations with rollback capability on failure.

## Pipeline Design

A typical migration pipeline has these stages:

1. Validate - lint and parse SQL migration files
2. Diff - preview changes against staging
3. Apply - run migrations on target environment
4. Verify - confirm schema matches expected state
5. Rollback (on failure) - revert if verification fails

## GitHub Actions Workflow

```yaml
# .github/workflows/clickhouse-migrations.yml
name: ClickHouse Schema Migrations

on:
  push:
    branches: [main]
    paths:
      - 'migrations/**'

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install migrate CLI
        run: |
          curl -L https://github.com/golang-migrate/migrate/releases/latest/download/migrate.linux-amd64.tar.gz \
            | tar xvz
          sudo mv migrate /usr/local/bin/

      - name: Validate SQL files
        run: |
          for f in migrations/*.sql; do
            clickhouse-format < "$f" > /dev/null || exit 1
          done

      - name: Check migration status
        run: |
          migrate \
            -path ./migrations \
            -database "${{ secrets.CLICKHOUSE_URL }}" \
            version

      - name: Apply migrations
        run: |
          migrate \
            -path ./migrations \
            -database "${{ secrets.CLICKHOUSE_URL }}" \
            up

      - name: Verify schema
        run: |
          clickhouse-client \
            --host ${{ secrets.CH_HOST }} \
            --query "SELECT name, type FROM system.columns WHERE table = 'events' ORDER BY name"
```

## Staging-First Pattern

Apply to staging first, run smoke tests, then promote to production:

```yaml
jobs:
  migrate-staging:
    environment: staging
    steps:
      - name: Apply to staging
        run: migrate -path ./migrations -database "${{ secrets.STAGING_CH_URL }}" up

      - name: Run smoke tests
        run: python tests/schema_smoke_test.py --host ${{ secrets.STAGING_CH_HOST }}

  migrate-production:
    needs: migrate-staging
    environment: production
    steps:
      - name: Apply to production
        run: migrate -path ./migrations -database "${{ secrets.PROD_CH_URL }}" up
```

## Smoke Test Script

```python
# tests/schema_smoke_test.py
import sys
from clickhouse_driver import Client

client = Client(sys.argv[sys.argv.index('--host') + 1])

# Verify expected columns exist
cols = {row[0] for row in client.execute(
    "SELECT name FROM system.columns WHERE table = 'events'"
)}

required = {'ts', 'user_id', 'event_type', 'session_id', 'country'}
missing = required - cols

if missing:
    print(f"FAIL: Missing columns: {missing}")
    sys.exit(1)

print("PASS: All required columns present")
```

## Automatic Rollback on Failure

```yaml
- name: Apply migrations
  id: apply
  run: migrate -path ./migrations -database "${{ secrets.CH_URL }}" up

- name: Rollback on failure
  if: failure() && steps.apply.outcome == 'failure'
  run: migrate -path ./migrations -database "${{ secrets.CH_URL }}" down 1
```

## Summary

Automating ClickHouse migrations in CI/CD with GitHub Actions, migrate CLI, and schema validation brings safety and consistency to production deployments. A staging-first approach with smoke tests minimizes the risk of breaking production schemas.
