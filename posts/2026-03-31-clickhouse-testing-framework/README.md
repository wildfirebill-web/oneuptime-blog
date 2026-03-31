# How to Build a ClickHouse Testing Framework

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Testing, Framework, Python, Integration Test

Description: Build a reusable testing framework for ClickHouse that covers schema validation, query correctness, and performance regression testing.

---

## Why You Need a ClickHouse Testing Framework

Ad hoc queries are easy to get wrong, and schema migrations can silently break downstream analytics. A structured testing framework lets you catch regressions in query logic, performance, and data contracts before they reach production.

## Project Structure

```text
clickhouse_tests/
  conftest.py          # shared fixtures
  test_schema.py       # DDL and column type checks
  test_queries.py      # query result correctness
  test_performance.py  # latency thresholds
  helpers.py           # reusable utilities
```

## Setting Up Fixtures with pytest

```python
# conftest.py
import pytest
import clickhouse_connect

@pytest.fixture(scope="session")
def ch_client():
    client = clickhouse_connect.get_client(
        host="localhost", port=8123,
        username="default", password=""
    )
    client.command("CREATE DATABASE IF NOT EXISTS test_db")
    yield client
    client.command("DROP DATABASE IF EXISTS test_db")
```

The `session`-scoped fixture creates an isolated test database once and tears it down after all tests complete.

## Schema Tests

```python
# test_schema.py
def test_events_table_exists(ch_client):
    result = ch_client.query(
        "SELECT count() FROM system.tables "
        "WHERE database='test_db' AND name='events'"
    )
    assert result.first_row[0] == 1

def test_events_column_types(ch_client):
    result = ch_client.query(
        "SELECT name, type FROM system.columns "
        "WHERE database='test_db' AND table='events'"
    )
    columns = {row[0]: row[1] for row in result.result_rows}
    assert columns["user_id"] == "UInt64"
    assert columns["event_time"] == "DateTime"
```

## Query Correctness Tests

```python
# test_queries.py
def test_daily_active_users(ch_client):
    ch_client.command("""
        INSERT INTO test_db.events (user_id, event_time, event_type)
        VALUES (1, now(), 'login'), (2, now(), 'login'), (1, now(), 'click')
    """)
    result = ch_client.query("""
        SELECT uniq(user_id) FROM test_db.events
        WHERE toDate(event_time) = today()
    """)
    assert result.first_row[0] == 2
```

## Performance Tests

```python
# test_performance.py
import time

def test_aggregation_latency(ch_client):
    start = time.monotonic()
    ch_client.query(
        "SELECT event_type, count() FROM test_db.events GROUP BY event_type"
    )
    elapsed = time.monotonic() - start
    assert elapsed < 2.0, f"Query too slow: {elapsed:.2f}s"
```

## Helpers for Test Data

```python
# helpers.py
def insert_test_events(client, n=1000):
    rows = [(i % 100, "login") for i in range(n)]
    client.insert(
        "test_db.events",
        rows,
        column_names=["user_id", "event_type"]
    )
```

## Running the Tests

```bash
pytest clickhouse_tests/ -v --tb=short
```

Add this to CI to run against a Docker-based ClickHouse instance on every pull request.

## Summary

A ClickHouse testing framework built on pytest gives you schema validation, query correctness checks, and performance regression gates. Shared fixtures keep setup DRY, while isolated test databases prevent interference between test runs.
