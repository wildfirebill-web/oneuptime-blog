# How to Load Dictionaries from MySQL Sources in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, MySQL, Integration, Data Loading

Description: Learn how to configure ClickHouse dictionaries to load dimension data directly from a MySQL database for real-time enrichment queries.

---

ClickHouse can load dictionary data from an external MySQL database. This is useful when your dimension tables (users, products, regions) live in an operational MySQL database and you want to use them for enrichment in ClickHouse analytics queries without duplicating the ETL pipeline.

## Creating a MySQL-Backed Dictionary

```sql
CREATE DICTIONARY user_dict
(
    user_id    UInt64,
    email      String,
    plan       String,
    country    String
)
PRIMARY KEY user_id
SOURCE(MYSQL(
    HOST     'mysql.example.com'
    PORT     3306
    USER     'clickhouse_reader'
    PASSWORD 'secret'
    DB       'app_db'
    TABLE    'users'
))
LAYOUT(HASHED())
LIFETIME(MIN 300 MAX 600);
```

ClickHouse issues `SELECT * FROM app_db.users` against the MySQL server on each reload.

## Using a Custom Query Instead of a Table

When you need only a subset of columns or rows, use `QUERY` instead of `TABLE`:

```sql
SOURCE(MYSQL(
    HOST     'mysql.example.com'
    PORT     3306
    USER     'clickhouse_reader'
    PASSWORD 'secret'
    DB       'app_db'
    QUERY    'SELECT user_id, email, plan, country FROM users WHERE active = 1'
))
```

## Connection Pooling Settings

```sql
SOURCE(MYSQL(
    HOST               'mysql.example.com'
    PORT               3306
    USER               'clickhouse_reader'
    PASSWORD           'secret'
    DB                 'app_db'
    TABLE              'users'
    CLOSE_CONNECTION   1
    SHARE_CONNECTION   0
))
```

- `CLOSE_CONNECTION 1` - closes the MySQL connection after each reload (lower resource usage)
- `SHARE_CONNECTION 0` - each dictionary gets its own connection

## Reloading the Dictionary

```sql
SYSTEM RELOAD DICTIONARY user_dict;
```

## Querying with the Dictionary

```sql
SELECT
    event_id,
    user_id,
    dictGet('user_dict', 'plan',    toUInt64(user_id)) AS plan,
    dictGet('user_dict', 'country', toUInt64(user_id)) AS country
FROM events
WHERE event_date = today()
LIMIT 20;
```

## Checking Status and Errors

```sql
SELECT name, element_count, status, last_exception
FROM system.dictionaries
WHERE name = 'user_dict';
```

## Replica Failover

Specify multiple MySQL replicas for automatic failover:

```sql
SOURCE(MYSQL(
    REPLICA(HOST 'mysql-primary.example.com' PORT 3306 PRIORITY 1)
    REPLICA(HOST 'mysql-replica.example.com' PORT 3306 PRIORITY 2)
    USER     'clickhouse_reader'
    PASSWORD 'secret'
    DB       'app_db'
    TABLE    'users'
))
```

## Security Note

Use a read-only MySQL user with SELECT privileges only on the required table. Store credentials in ClickHouse's `users.xml` or environment substitution rather than in plain SQL if possible.

## Summary

MySQL-backed dictionaries let ClickHouse pull dimension data from your operational database on a configurable refresh schedule. By combining the `MYSQL` source with `HASHED` layout and `LIFETIME` settings, you get fast in-memory lookups that stay in sync with your MySQL tables without a separate ETL process.
