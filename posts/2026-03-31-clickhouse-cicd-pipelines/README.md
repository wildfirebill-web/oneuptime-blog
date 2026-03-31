# How to Set Up ClickHouse in CI/CD Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CI/CD, DevOps, Testing, Automation

Description: Learn how to run ClickHouse in CI/CD pipelines for integration testing and schema migration validation using Docker service containers.

---

## Why Run ClickHouse in CI/CD

Running integration tests against a real ClickHouse instance in CI prevents regressions from schema changes, query rewrites, and data model migrations. CI-managed ClickHouse instances are ephemeral and isolated per pipeline run.

## GitHub Actions with ClickHouse Service Container

```yaml
name: Integration Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      clickhouse:
        image: clickhouse/clickhouse-server:24.3
        ports:
          - 8123:8123
          - 9000:9000
        options: >-
          --health-cmd "wget -qO- http://localhost:8123/ping || exit 1"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Wait for ClickHouse
        run: |
          until wget -qO- http://localhost:8123/ping; do sleep 1; done

      - name: Run schema migrations
        run: python scripts/migrate.py --host localhost --port 8123

      - name: Run integration tests
        run: pytest tests/integration/ -v
        env:
          CLICKHOUSE_HOST: localhost
          CLICKHOUSE_PORT: 8123
```

## GitLab CI with ClickHouse Service

```yaml
integration-test:
  image: python:3.12-slim
  services:
    - name: clickhouse/clickhouse-server:24.3
      alias: clickhouse
  variables:
    CLICKHOUSE_HOST: clickhouse
    CLICKHOUSE_PORT: 8123
  before_script:
    - pip install -r requirements.txt
    - |
      until wget -qO- http://$CLICKHOUSE_HOST:$CLICKHOUSE_PORT/ping; do
        echo "Waiting for ClickHouse..."; sleep 2
      done
  script:
    - python scripts/migrate.py
    - pytest tests/integration/ -v
```

## Running Schema Validation in CI

```python
# scripts/migrate.py
import clickhouse_connect
import os
import glob

client = clickhouse_connect.get_client(
    host=os.getenv('CLICKHOUSE_HOST', 'localhost'),
    port=int(os.getenv('CLICKHOUSE_PORT', 8123)),
)

for migration_file in sorted(glob.glob('migrations/*.sql')):
    print(f'Running {migration_file}')
    with open(migration_file) as f:
        client.command(f.read())
print('All migrations applied')
```

## Migrations Directory

```text
migrations/
  001_create_events.sql
  002_add_revenue_column.sql
  003_create_materialized_view.sql
```

## Parallel Test Isolation

Each test worker should use a different database to avoid conflicts:

```python
import os
import pytest
import clickhouse_connect

@pytest.fixture(scope='session')
def ch_client(worker_id):
    db = f'test_{worker_id}'
    client = clickhouse_connect.get_client(host=os.getenv('CLICKHOUSE_HOST'))
    client.command(f'CREATE DATABASE IF NOT EXISTS {db}')
    yield client
    client.command(f'DROP DATABASE IF EXISTS {db}')
```

## Summary

Run ClickHouse as a CI service container in GitHub Actions or GitLab CI, then apply schema migrations and execute integration tests against it. Use health checks to wait for ClickHouse readiness before tests start. Isolate parallel test workers with separate databases to prevent flaky failures from shared state.
