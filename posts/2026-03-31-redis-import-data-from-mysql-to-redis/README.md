# How to Import Data from MySQL to Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, MySQL, Import, Caching, ETL

Description: Learn how to import MySQL table rows into Redis as hashes, strings, and sorted sets using Python for cache warming and fast read patterns.

---

Importing MySQL data into Redis is a common cache-warming strategy. Instead of letting your cache fill up gradually under load (causing a cold start), you pre-populate Redis from MySQL before traffic hits. This guide covers the main patterns with Python.

## Setup

```python
import redis
import mysql.connector
import json

r = redis.Redis(host="localhost", port=6379, password="redis-password", decode_responses=True)

mysql_conn = mysql.connector.connect(
    host="mysql-host",
    user="dbuser",
    password="dbpassword",
    database="mydb"
)
cur = mysql_conn.cursor(dictionary=True)  # returns rows as dicts
```

## Import Table Rows as Redis Hashes

```python
def import_table_as_hashes(table, id_column, key_prefix, ttl_seconds=3600, limit=None):
    query = f"SELECT * FROM {table}"
    if limit:
        query += f" LIMIT {limit}"

    cur.execute(query)
    rows = cur.fetchall()

    pipeline = r.pipeline(transaction=False)
    count = 0

    for row in rows:
        row_id = str(row[id_column])
        redis_key = f"{key_prefix}:{row_id}"

        # Convert all values to strings (Redis hash values must be strings)
        string_row = {k: str(v) if v is not None else "" for k, v in row.items()}

        pipeline.hset(redis_key, mapping=string_row)
        if ttl_seconds:
            pipeline.expire(redis_key, ttl_seconds)

        count += 1
        if count % 500 == 0:
            pipeline.execute()
            print(f"Imported {count} rows from {table}...")

    pipeline.execute()
    print(f"Done: {count} rows from '{table}' imported")

import_table_as_hashes("users", "id", "user")
import_table_as_hashes("products", "product_id", "product")
```

## Import with Pagination for Large Tables

```python
def import_large_table_paginated(table, id_column, key_prefix, page_size=1000, ttl_seconds=3600):
    offset = 0
    total = 0

    while True:
        cur.execute(f"""
            SELECT * FROM {table}
            ORDER BY {id_column}
            LIMIT %s OFFSET %s
        """, (page_size, offset))

        rows = cur.fetchall()
        if not rows:
            break

        pipeline = r.pipeline(transaction=False)
        for row in rows:
            row_id = str(row[id_column])
            redis_key = f"{key_prefix}:{row_id}"
            string_row = {k: str(v) if v is not None else "" for k, v in row.items()}
            pipeline.hset(redis_key, mapping=string_row)
            if ttl_seconds:
                pipeline.expire(redis_key, ttl_seconds)

        pipeline.execute()
        total += len(rows)
        offset += page_size
        print(f"Imported {total} rows...")

    print(f"Done: {total} total rows from {table}")
```

## Import a Single Column as String Keys

For simple key-value lookups like feature flags or short-lived tokens:

```python
def import_config_as_strings(ttl_seconds=None):
    cur.execute("SELECT config_key, config_value FROM app_config WHERE enabled = 1")
    rows = cur.fetchall()

    pipeline = r.pipeline(transaction=False)
    for row in rows:
        key = f"config:{row['config_key']}"
        pipeline.set(key, row['config_value'])
        if ttl_seconds:
            pipeline.expire(key, ttl_seconds)

    pipeline.execute()
    print(f"Done: {len(rows)} config values imported")
```

## Import Ranked Data as Sorted Set

```python
def import_leaderboard(mysql_table="scores", redis_key="leaderboard:global", ttl_seconds=86400):
    cur.execute(f"""
        SELECT user_id, username, total_points
        FROM {mysql_table}
        ORDER BY total_points DESC
        LIMIT 10000
    """)
    rows = cur.fetchall()

    score_map = {}
    for row in rows:
        member = f"{row['user_id']}:{row['username']}"
        score_map[member] = float(row['total_points'])

    if score_map:
        r.zadd(redis_key, score_map)
        if ttl_seconds:
            r.expire(redis_key, ttl_seconds)

    print(f"Done: {len(score_map)} leaderboard entries imported")

import_leaderboard()
```

## Import JSON/TEXT Columns as Redis Strings

```python
def import_json_column(table, id_column, json_column, key_prefix, ttl_seconds=1800):
    cur.execute(f"SELECT {id_column}, {json_column} FROM {table} WHERE {json_column} IS NOT NULL")
    rows = cur.fetchall()

    pipeline = r.pipeline(transaction=False)
    for row in rows:
        key = f"{key_prefix}:{row[id_column]}"
        value = row[json_column]
        # If the value is already a JSON string, store it directly
        pipeline.set(key, value if isinstance(value, str) else json.dumps(value))
        if ttl_seconds:
            pipeline.expire(key, ttl_seconds)

    pipeline.execute()
    print(f"Done: {len(rows)} JSON values imported")
```

## Verify the Import

```python
def verify_import(table, id_column, key_prefix, sample_size=10):
    cur.execute(f"SELECT {id_column} FROM {table} ORDER BY RAND() LIMIT %s", (sample_size,))
    sample_ids = [str(row[id_column]) for row in cur.fetchall()]

    found = 0
    for row_id in sample_ids:
        redis_key = f"{key_prefix}:{row_id}"
        exists = r.exists(redis_key)
        status = "FOUND" if exists else "MISSING"
        print(f"  {redis_key}: {status}")
        if exists:
            found += 1

    print(f"Validation: {found}/{sample_size} keys found in Redis")

verify_import("users", "id", "user")
```

## Summary

Importing MySQL rows into Redis is an effective cache-warming strategy. Use `dictionary=True` cursor to get rows as dicts, then write them as Redis hashes with `pipeline.hset()`. Paginate large tables to avoid memory pressure, apply TTLs for automatic cache expiry, and validate by spot-checking random keys after import.
