# How to Import Data from PostgreSQL to Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PostgreSQL, Import, ETL, Caching

Description: Learn how to import data from PostgreSQL to Redis for caching, using Python to read rows and write them as Redis hashes, strings, and sorted sets.

---

Importing data from PostgreSQL into Redis is a common pattern for warming a cache, pre-loading session data, or building leaderboards from relational data. This guide shows how to read from PostgreSQL and write to appropriate Redis data types.

## Setup

```python
import redis
import psycopg2
import json

r = redis.Redis(host="localhost", port=6379, password="redis-password", decode_responses=True)
pg = psycopg2.connect("postgresql://user:password@localhost:5432/mydb")
cur = pg.cursor()
```

## Import Table Rows as Redis Hashes

The most common pattern: cache user profiles or config records as Redis hashes.

```python
def import_users_to_redis(ttl_seconds=3600):
    cur.execute("SELECT id, name, email, role, created_at::text FROM users")
    rows = cur.fetchall()
    columns = [desc[0] for desc in cur.description]

    pipeline = r.pipeline(transaction=False)
    count = 0

    for row in rows:
        user_data = dict(zip(columns, [str(v) if v is not None else "" for v in row]))
        redis_key = f"user:{user_data['id']}"

        pipeline.hset(redis_key, mapping=user_data)
        if ttl_seconds:
            pipeline.expire(redis_key, ttl_seconds)

        count += 1
        if count % 500 == 0:
            pipeline.execute()
            print(f"Imported {count} users...")

    pipeline.execute()
    print(f"Done: {count} users imported to Redis")

import_users_to_redis()
```

## Import Single-Value Rows as Strings

For simple key-value lookups like feature flags or config values:

```python
def import_config_to_redis(ttl_seconds=None):
    cur.execute("SELECT config_key, config_value FROM app_config WHERE active = true")
    rows = cur.fetchall()

    pipeline = r.pipeline(transaction=False)
    count = 0

    for key, value in rows:
        redis_key = f"config:{key}"
        pipeline.set(redis_key, value)
        if ttl_seconds:
            pipeline.expire(redis_key, ttl_seconds)
        count += 1

    pipeline.execute()
    print(f"Done: {count} config values imported")

import_config_to_redis()
```

## Import Scores/Rankings as Sorted Set

For leaderboards or ranked lists:

```python
def import_leaderboard_to_redis(leaderboard_key="leaderboard:global", ttl_seconds=86400):
    cur.execute("""
        SELECT user_id, username, total_score
        FROM user_scores
        ORDER BY total_score DESC
        LIMIT 10000
    """)
    rows = cur.fetchall()

    # Build score mapping: {member: score}
    score_mapping = {}
    for user_id, username, score in rows:
        member = f"{user_id}:{username}"
        score_mapping[member] = float(score)

    if score_mapping:
        r.zadd(leaderboard_key, score_mapping)
        if ttl_seconds:
            r.expire(leaderboard_key, ttl_seconds)

    print(f"Done: {len(score_mapping)} leaderboard entries imported")

import_leaderboard_to_redis()
```

## Import with Pagination for Large Tables

```python
def import_large_table_to_redis(table, id_column, page_size=1000, ttl_seconds=3600):
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

        columns = [desc[0] for desc in cur.description]
        pipeline = r.pipeline(transaction=False)

        for row in rows:
            data = dict(zip(columns, [str(v) if v is not None else "" for v in row]))
            key = f"{table}:{data[id_column]}"
            pipeline.hset(key, mapping=data)
            if ttl_seconds:
                pipeline.expire(key, ttl_seconds)

        pipeline.execute()
        total += len(rows)
        offset += page_size
        print(f"Imported {total} rows from {table}...")

    print(f"Done: {total} rows imported")
```

## Import JSON Columns

If your PostgreSQL table has JSONB columns:

```python
def import_json_documents_to_redis(ttl_seconds=1800):
    cur.execute("SELECT doc_id, metadata FROM documents WHERE active = true")
    rows = cur.fetchall()

    pipeline = r.pipeline(transaction=False)
    for doc_id, metadata in rows:
        key = f"doc:{doc_id}"
        # Store as JSON string
        pipeline.set(key, json.dumps(metadata) if isinstance(metadata, dict) else metadata)
        if ttl_seconds:
            pipeline.expire(key, ttl_seconds)

    pipeline.execute()
    print(f"Done: {len(rows)} documents imported")
```

## Verify the Import

```python
def verify_import(table, id_column, sample_ids):
    for sid in sample_ids:
        cur.execute(f"SELECT * FROM {table} WHERE {id_column} = %s", (sid,))
        pg_row = cur.fetchone()
        pg_columns = [desc[0] for desc in cur.description]
        pg_data = dict(zip(pg_columns, pg_row)) if pg_row else {}

        redis_key = f"{table}:{sid}"
        redis_data = r.hgetall(redis_key)

        match = all(str(pg_data.get(k, "")) == v for k, v in redis_data.items())
        print(f"ID {sid}: {'MATCH' if match else 'MISMATCH'}")

verify_import("users", "id", ["1001", "1002", "1003"])
```

## Summary

Importing from PostgreSQL to Redis involves reading rows with `psycopg2` and writing to Redis using the appropriate data type: hashes for multi-field records, strings for simple values, and sorted sets for ranked data. Use pipelines to batch writes and page through large tables to avoid memory pressure.
