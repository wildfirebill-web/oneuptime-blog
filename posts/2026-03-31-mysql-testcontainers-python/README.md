# How to Use MySQL Testcontainers in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Testcontainers, Python, Integration Test

Description: Learn how to use the testcontainers-python library to launch a real MySQL Docker container in pytest, manage schema setup, and isolate tests with transaction rollbacks.

---

## Why Testcontainers in Python Tests

Testcontainers for Python launches a real MySQL Docker container from within your pytest session, applies your schema, and tears down cleanly when tests are done. This gives Python applications the same isolation and reproducibility benefits as any other language without requiring a pre-configured test server.

## Installation

```bash
pip install testcontainers[mysql] pymysql pytest
```

Docker must be available on the host or CI agent.

## Conftest with Session-Scoped Container

Define the container once per test session in `conftest.py`:

```python
# tests/conftest.py
import pytest
import pymysql
from testcontainers.mysql import MySqlContainer

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS users (
    id         BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
    email      VARCHAR(200) NOT NULL UNIQUE,
    role       VARCHAR(50)  NOT NULL DEFAULT 'USER',
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

CREATE TABLE IF NOT EXISTS orders (
    id          BIGINT        NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT        NOT NULL,
    total       DECIMAL(10,2) NOT NULL,
    status      VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
    INDEX idx_user (user_id)
) ENGINE=InnoDB;
"""

@pytest.fixture(scope="session")
def mysql_container():
    with MySqlContainer("mysql:8.0") as container:
        yield container

@pytest.fixture(scope="session")
def db_connection(mysql_container):
    conn = pymysql.connect(
        host=mysql_container.get_container_host_ip(),
        port=int(mysql_container.get_exposed_port(3306)),
        user="test",
        password="test",
        database="test",
        autocommit=False
    )
    with conn.cursor() as cur:
        for statement in SCHEMA_SQL.split(";"):
            if statement.strip():
                cur.execute(statement)
    conn.commit()
    yield conn
    conn.close()

@pytest.fixture(autouse=True)
def rollback_after_test(db_connection):
    """Roll back all changes after each test for isolation."""
    yield
    db_connection.rollback()
```

## Writing Tests

```python
# tests/test_orders.py
def test_create_order(db_connection):
    with db_connection.cursor() as cur:
        cur.execute(
            "INSERT INTO users (email, role) VALUES (%s, %s)",
            ("test@example.com", "USER")
        )
        user_id = cur.lastrowid

        cur.execute(
            "INSERT INTO orders (user_id, total, status) VALUES (%s, %s, %s)",
            (user_id, 59.99, "PENDING")
        )
        order_id = cur.lastrowid

        cur.execute("SELECT status FROM orders WHERE id = %s", (order_id,))
        row = cur.fetchone()

    assert row[0] == "PENDING"


def test_update_order_status(db_connection):
    with db_connection.cursor() as cur:
        cur.execute(
            "INSERT INTO users (email) VALUES (%s)", ("user2@example.com",)
        )
        user_id = cur.lastrowid
        cur.execute(
            "INSERT INTO orders (user_id, total) VALUES (%s, %s)",
            (user_id, 25.00)
        )
        order_id = cur.lastrowid

        cur.execute(
            "UPDATE orders SET status = %s WHERE id = %s",
            ("COMPLETED", order_id)
        )
        cur.execute("SELECT status FROM orders WHERE id = %s", (order_id,))
        row = cur.fetchone()

    assert row[0] == "COMPLETED"


def test_user_not_found_returns_none(db_connection):
    with db_connection.cursor() as cur:
        cur.execute("SELECT * FROM users WHERE id = %s", (99999,))
        row = cur.fetchone()
    assert row is None
```

## Loading Schema from a File

```python
import os

@pytest.fixture(scope="session")
def db_connection(mysql_container):
    conn = pymysql.connect(
        host=mysql_container.get_container_host_ip(),
        port=int(mysql_container.get_exposed_port(3306)),
        user="test", password="test", database="test",
        autocommit=False
    )
    schema_path = os.path.join(os.path.dirname(__file__), "../db/schema.sql")
    with open(schema_path) as f:
        schema = f.read()
    with conn.cursor() as cur:
        for stmt in schema.split(";"):
            if stmt.strip():
                cur.execute(stmt)
    conn.commit()
    yield conn
    conn.close()
```

## Running the Tests

```bash
pytest tests/ -v
```

The first run downloads the `mysql:8.0` image. Subsequent runs reuse the cached image and start in seconds.

## Summary

Testcontainers for Python integrates cleanly with pytest fixtures to provide a real MySQL container for integration tests. Use a session-scoped fixture for the container and connection, an autouse fixture for rollback isolation, and `pymysql` for direct SQL execution. The result is fully isolated, reproducible integration tests that require no external database infrastructure.
