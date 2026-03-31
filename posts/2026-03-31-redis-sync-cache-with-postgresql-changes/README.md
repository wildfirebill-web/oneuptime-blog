# How to Sync Redis Cache with PostgreSQL Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PostgreSQL, Cache Sync, Change Data Capture, Debezium, Invalidation

Description: Sync Redis cache with PostgreSQL changes using database triggers, LISTEN/NOTIFY, and Change Data Capture with Debezium to keep cached data consistent with the source database.

---

## The Cache Consistency Problem

When data changes in PostgreSQL, the Redis cache can serve stale data until the TTL expires. For data that changes frequently, this is unacceptable. There are three solutions:

1. **Application-level invalidation** - delete the cache key when your code updates the database
2. **PostgreSQL LISTEN/NOTIFY** - database sends a notification on change
3. **Change Data Capture (CDC)** - capture database changes at the WAL level

## Approach 1 - Application-Level Invalidation

The simplest approach: invalidate the cache in the same transaction as the database update.

```python
import redis
import psycopg2
import json

r = redis.Redis(decode_responses=True)
pg = psycopg2.connect(host="localhost", database="myapp", user="postgres", password="password")

def update_user(user_id: int, updates: dict):
    with pg.cursor() as cursor:
        set_clause = ", ".join(f"{k} = %s" for k in updates.keys())
        values = list(updates.values()) + [user_id]
        cursor.execute(f"UPDATE users SET {set_clause} WHERE id = %s", values)
    pg.commit()

    # Invalidate cache immediately after successful commit
    r.delete(f"user:{user_id}")
    # Also invalidate any list/query caches for users
    for key in r.scan_iter("users:list:*"):
        r.delete(key)

def delete_user(user_id: int):
    with pg.cursor() as cursor:
        cursor.execute("DELETE FROM users WHERE id = %s", (user_id,))
    pg.commit()

    r.delete(f"user:{user_id}")
```

## Approach 2 - PostgreSQL LISTEN/NOTIFY

Use PostgreSQL's built-in LISTEN/NOTIFY to broadcast changes:

### PostgreSQL Trigger

```sql
-- Create notification function
CREATE OR REPLACE FUNCTION notify_cache_invalidation()
RETURNS TRIGGER AS $$
DECLARE
  payload TEXT;
BEGIN
  payload := json_build_object(
    'table', TG_TABLE_NAME,
    'operation', TG_OP,
    'id', COALESCE(NEW.id, OLD.id)
  )::TEXT;
  PERFORM pg_notify('cache_invalidation', payload);
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Attach trigger to users table
CREATE TRIGGER users_cache_invalidation
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION notify_cache_invalidation();
```

### Python Listener

```python
import psycopg2
import psycopg2.extensions
import select
import json
import redis

r = redis.Redis(decode_responses=True)
conn = psycopg2.connect(host="localhost", database="myapp", user="postgres", password="password")
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

cursor = conn.cursor()
cursor.execute("LISTEN cache_invalidation;")
print("Listening for cache invalidation events...")

while True:
    if select.select([conn], [], [], 5) == ([], [], []):
        continue

    conn.poll()
    while conn.notifies:
        notify = conn.notifies.pop(0)
        payload = json.loads(notify.payload)

        table = payload["table"]
        record_id = payload["id"]
        operation = payload["operation"]

        # Invalidate the specific record
        cache_key = f"{table}:{record_id}"
        r.delete(cache_key)

        # Invalidate related list caches
        for key in r.scan_iter(f"{table}:list:*"):
            r.delete(key)

        print(f"Invalidated {operation} on {cache_key}")
```

## Approach 3 - Change Data Capture with Debezium

Debezium reads PostgreSQL's Write-Ahead Log (WAL) and streams changes to Kafka, which a consumer uses to update Redis.

### Configure PostgreSQL for WAL Replication

```bash
# postgresql.conf
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
```

### Debezium Connector Config

```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "password",
    "database.dbname": "myapp",
    "database.server.name": "myapp",
    "table.include.list": "public.users,public.products",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot"
  }
}
```

### Python Kafka Consumer for Cache Updates

```python
from kafka import KafkaConsumer
import json
import redis

r = redis.Redis(decode_responses=True)

consumer = KafkaConsumer(
    "myapp.public.users",
    "myapp.public.products",
    bootstrap_servers=["kafka:9092"],
    value_deserializer=lambda m: json.loads(m.decode("utf-8"))
)

def handle_change(topic, event):
    table = topic.split(".")[-1]
    operation = event.get("op")
    record_id = (event.get("after") or event.get("before") or {}).get("id")

    if not record_id:
        return

    if operation in ("u", "d"):
        r.delete(f"{table}:{record_id}")
        print(f"Invalidated {table}:{record_id} (op={operation})")
    elif operation == "c":
        # Optionally write-through on create
        after = event.get("after")
        if after:
            r.set(f"{table}:{record_id}", json.dumps(after), ex=3600)
            print(f"Cached new {table}:{record_id}")

for message in consumer:
    handle_change(message.topic, message.value)
```

## Cache Warm-Up After Sync

```python
def warm_cache_from_postgres(table: str, id_column: str = "id"):
    with pg.cursor() as cur:
        cur.execute(f"SELECT * FROM {table} LIMIT 1000")
        cols = [d[0] for d in cur.description]
        rows = cur.fetchall()

    pipe = r.pipeline()
    for row in rows:
        record = dict(zip(cols, row))
        pipe.set(f"{table}:{record[id_column]}", json.dumps(record, default=str), ex=3600)
    pipe.execute()
    print(f"Warmed {len(rows)} {table} records")
```

## Summary

Syncing Redis cache with PostgreSQL uses one of three strategies based on consistency requirements: application-level invalidation (simplest, requires discipline), PostgreSQL LISTEN/NOTIFY (near-real-time without extra infrastructure), or Debezium CDC (reliable at-scale change capture via WAL). CDC is the most reliable for high-throughput systems since it captures every change including those from migrations, background jobs, and direct database access.
