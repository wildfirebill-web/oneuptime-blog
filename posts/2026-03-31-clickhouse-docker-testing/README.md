# How to Run ClickHouse Tests in Docker Containers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Testing, Integration Test, CI/CD, Testcontainers

Description: Learn how to run ClickHouse integration tests using Docker containers in CI/CD pipelines with ephemeral instances and automated schema setup.

---

Docker makes it straightforward to run ClickHouse integration tests against a real instance in CI/CD pipelines. Each test run gets a fresh, isolated ClickHouse container, eliminating shared-state issues between test suites.

## Basic Test Setup with Docker Compose

Create a `docker-compose.test.yml` for isolated test infrastructure:

```yaml
version: '3.8'

services:
  clickhouse-test:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8124:8123"
      - "9001:9000"
    environment:
      CLICKHOUSE_DB: test
      CLICKHOUSE_USER: test
      CLICKHOUSE_PASSWORD: test
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8123/ping"]
      interval: 5s
      timeout: 3s
      retries: 10
```

## Running Tests in CI (GitHub Actions)

```yaml
name: Integration Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start ClickHouse
        run: docker compose -f docker-compose.test.yml up -d --wait

      - name: Run migrations
        run: |
          clickhouse-client --host localhost --port 9001 \
            --user test --password test \
            --multiquery < schema/migrations.sql

      - name: Run integration tests
        run: npm test
        env:
          CLICKHOUSE_HOST: localhost
          CLICKHOUSE_PORT: 9001
          CLICKHOUSE_USER: test
          CLICKHOUSE_PASSWORD: test
          CLICKHOUSE_DB: test

      - name: Stop ClickHouse
        if: always()
        run: docker compose -f docker-compose.test.yml down -v
```

## Using Testcontainers (Python)

```python
from testcontainers.clickhouse import ClickHouseContainer
import clickhouse_driver

def test_insert_and_query():
    with ClickHouseContainer("clickhouse/clickhouse-server:24.3") as ch:
        client = clickhouse_driver.Client(
            host=ch.get_container_host_ip(),
            port=ch.get_exposed_port(9000)
        )

        client.execute("CREATE TABLE test (id UInt64) ENGINE = MergeTree ORDER BY id")
        client.execute("INSERT INTO test VALUES", [(1,), (2,), (3,)])

        result = client.execute("SELECT count() FROM test")
        assert result[0][0] == 3
```

## Schema Seeding Script

```bash
#!/bin/bash
set -e

echo "Waiting for ClickHouse..."
until clickhouse-client --host "${CH_HOST:-localhost}" --query "SELECT 1" > /dev/null 2>&1; do
  sleep 1
done

echo "Running schema migrations..."
for f in schema/*.sql; do
  echo "Applying $f"
  clickhouse-client --host "${CH_HOST:-localhost}" --multiquery < "$f"
done

echo "Schema ready."
```

## Testing TTL and Merges

```sql
-- Force merges for TTL/dedup testing
OPTIMIZE TABLE test_table FINAL;

-- Force TTL cleanup
ALTER TABLE test_table MATERIALIZE TTL;
```

## Summary

Run ClickHouse tests in Docker by starting an ephemeral container, applying schema migrations, running your test suite, and tearing down with `docker compose down -v`. Use Testcontainers for programmatic container lifecycle management in unit/integration tests, and cache Docker images in CI to reduce startup time.
