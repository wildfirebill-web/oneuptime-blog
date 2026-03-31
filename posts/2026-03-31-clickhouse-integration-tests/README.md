# How to Write Integration Tests for ClickHouse Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Integration Test, Testing, Python, CI/CD

Description: Learn how to write integration tests for ClickHouse queries using Python's clickhouse-connect client with a local ClickHouse instance for reliable test isolation.

---

## Why Integration Test ClickHouse Queries

Unit tests with mocked SQL clients don't catch schema mismatches, type errors, or aggregation bugs. Integration tests run queries against a real ClickHouse instance, ensuring your application logic produces the correct results.

## Test Setup with pytest and clickhouse-connect

```bash
pip install pytest clickhouse-connect pytest-docker
```

## conftest.py - Test Fixtures

```python
import pytest
import clickhouse_connect

@pytest.fixture(scope='session')
def ch_client():
    client = clickhouse_connect.get_client(
        host='localhost',
        port=8123,
        username='default',
        password='',
        database='test_db',
    )
    # Create schema
    client.command('CREATE DATABASE IF NOT EXISTS test_db')
    client.command('''
        CREATE TABLE IF NOT EXISTS test_db.events (
            user_id   UInt64,
            event_type LowCardinality(String),
            revenue   Float64,
            event_date Date
        ) ENGINE = MergeTree()
        ORDER BY (user_id, event_date)
    ''')
    yield client
    client.command('DROP TABLE IF EXISTS test_db.events')
```

## Writing Tests

```python
def test_revenue_aggregation(ch_client):
    ch_client.command("TRUNCATE TABLE test_db.events")
    ch_client.insert('test_db.events', [
        [1, 'purchase', 100.0, '2025-01-01'],
        [1, 'purchase', 200.0, '2025-01-01'],
        [2, 'purchase',  50.0, '2025-01-01'],
    ], column_names=['user_id', 'event_type', 'revenue', 'event_date'])

    result = ch_client.query('''
        SELECT user_id, sum(revenue) AS total
        FROM test_db.events
        WHERE event_type = 'purchase'
        GROUP BY user_id
        ORDER BY user_id
    ''')
    rows = result.result_rows
    assert rows[0] == (1, 300.0)
    assert rows[1] == (2, 50.0)

def test_empty_result_on_no_data(ch_client):
    ch_client.command("TRUNCATE TABLE test_db.events")
    result = ch_client.query('SELECT count() FROM test_db.events')
    assert result.result_rows[0][0] == 0

def test_materialized_view_updates(ch_client):
    ch_client.command('''
        CREATE MATERIALIZED VIEW IF NOT EXISTS test_db.daily_revenue
        ENGINE = SummingMergeTree()
        ORDER BY (event_date)
        AS SELECT event_date, sum(revenue) AS total
        FROM test_db.events
        GROUP BY event_date
    ''')
    ch_client.insert('test_db.events', [
        [1, 'purchase', 75.0, '2025-06-01'],
    ], column_names=['user_id', 'event_type', 'revenue', 'event_date'])

    result = ch_client.query('''
        SELECT event_date, sum(total) AS total
        FROM test_db.daily_revenue
        GROUP BY event_date
    ''')
    assert result.result_rows[0][1] == 75.0
```

## Running Tests

```bash
pytest tests/ -v
pytest tests/ -v -k "test_revenue"
```

## Running ClickHouse for Tests with Docker

```bash
docker run -d --name ch-test -p 8123:8123 clickhouse/clickhouse-server:24.3
pytest tests/
docker rm -f ch-test
```

## Summary

Integration tests for ClickHouse use `clickhouse-connect` with a real ClickHouse instance - either a local install or a Docker container. Use pytest fixtures to create and tear down test schemas, insert controlled data before each test, and assert on query results. This catches schema evolution bugs and aggregation logic errors that mock-based unit tests miss.
