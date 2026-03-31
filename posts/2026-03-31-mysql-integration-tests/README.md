# How to Use MySQL in Integration Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Integration Test, Docker, Testing

Description: Learn how to run integration tests against a real MySQL instance using Docker, transaction rollbacks for isolation, and seed data management with practical examples.

---

## Why Integration Tests Need a Real MySQL Instance

Unit tests mock the database, but integration tests should run against the same MySQL version used in production. This catches schema migration issues, collation mismatches, and query behavior differences that mocks cannot reproduce.

## Spin Up MySQL with Docker Compose

Create a `docker-compose.test.yml` for your test environment:

```yaml
version: "3.8"
services:
  mysql-test:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: app_test
      MYSQL_USER: testuser
      MYSQL_PASSWORD: testpass
    ports:
      - "3307:3306"
    tmpfs:
      - /var/lib/mysql   # run on tmpfs for speed
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 10
```

Start and wait for healthy:

```bash
docker compose -f docker-compose.test.yml up -d --wait
```

## Apply Schema Migrations

```bash
mysql -h 127.0.0.1 -P 3307 -u testuser -ptestpass app_test \
  < migrations/001_create_orders.sql
```

## Node.js Integration Test Example

```javascript
const mysql = require('mysql2/promise');
const assert = require('assert');

let conn;

before(async () => {
  conn = await mysql.createConnection({
    host: '127.0.0.1',
    port: 3307,
    user: 'testuser',
    password: 'testpass',
    database: 'app_test'
  });
});

beforeEach(async () => {
  await conn.beginTransaction();
});

afterEach(async () => {
  await conn.rollback(); // isolate each test
});

after(async () => {
  await conn.end();
});

it('inserts an order and retrieves it', async () => {
  await conn.execute(
    'INSERT INTO orders (customer_id, total, status) VALUES (?, ?, ?)',
    [42, 99.99, 'PENDING']
  );
  const [rows] = await conn.execute(
    'SELECT * FROM orders WHERE customer_id = ?', [42]
  );
  assert.strictEqual(rows.length, 1);
  assert.strictEqual(rows[0].status, 'PENDING');
});
```

## Python Integration Test Example

```python
import pytest
import pymysql

@pytest.fixture(scope="module")
def db():
    conn = pymysql.connect(
        host="127.0.0.1", port=3307,
        user="testuser", password="testpass",
        database="app_test", autocommit=False
    )
    yield conn
    conn.close()

@pytest.fixture(autouse=True)
def rollback(db):
    yield
    db.rollback()

def test_create_order(db):
    with db.cursor() as cur:
        cur.execute(
            "INSERT INTO orders (customer_id, total, status) VALUES (%s, %s, %s)",
            (99, 49.99, "PENDING")
        )
        cur.execute("SELECT status FROM orders WHERE customer_id = %s", (99,))
        row = cur.fetchone()
    assert row[0] == "PENDING"
```

## Seed Data Management

Use a SQL seed file run before each test suite:

```sql
-- seeds/test_data.sql
TRUNCATE TABLE orders;
TRUNCATE TABLE customers;

INSERT INTO customers (id, name, email) VALUES
  (1, 'Test User A', 'a@test.com'),
  (2, 'Test User B', 'b@test.com');
```

```bash
mysql -h 127.0.0.1 -P 3307 -u testuser -ptestpass app_test \
  < seeds/test_data.sql
```

## Running in CI with GitHub Actions

```yaml
jobs:
  integration-tests:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: app_test
          MYSQL_USER: testuser
          MYSQL_PASSWORD: testpass
          MYSQL_ROOT_PASSWORD: rootpass
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=5s --health-timeout=3s --health-retries=10
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: mysql -h 127.0.0.1 -u testuser -ptestpass app_test < migrations/schema.sql
      - run: npm test
```

## Summary

Integration tests against MySQL should use a dedicated Docker instance, apply real schema migrations, and isolate each test with transaction rollbacks. Use `tmpfs` mounts to speed up the MySQL container, seed data files for a consistent baseline, and GitHub Actions service containers for CI pipelines. This combination gives you high confidence that your application behaves correctly against the real database engine.
