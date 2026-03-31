# How to Use Redis with PostgreSQL for Full-Text Search

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, PostgreSQL, Full-Text Search, Cache, Performance

Description: Learn how to use Redis to cache PostgreSQL full-text search results, reducing repeated expensive tsvector queries and improving search response times.

---

PostgreSQL has excellent built-in full-text search via `tsvector` and `tsquery`, but repeated identical searches can strain your database under load. Redis caches search results so repeated queries are served without hitting PostgreSQL at all.

## PostgreSQL Full-Text Search Setup

```sql
-- Add tsvector column and GIN index
ALTER TABLE products ADD COLUMN search_vector tsvector;

UPDATE products
SET search_vector = to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, ''));

CREATE INDEX products_search_idx ON products USING GIN(search_vector);

-- Keep it updated with a trigger
CREATE FUNCTION products_search_trigger() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := to_tsvector('english',
    coalesce(NEW.name, '') || ' ' || coalesce(NEW.description, ''));
  RETURN NEW;
END
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_update
  BEFORE INSERT OR UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION products_search_trigger();
```

## Cached Search Function

```python
import redis
import psycopg2
import json
import hashlib

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
conn = psycopg2.connect("postgresql://app:secret@localhost/myapp")

SEARCH_CACHE_TTL = 60  # 1 minute for search results

def cache_key_for_search(query: str, limit: int, offset: int) -> str:
    raw = f"search:{query}:{limit}:{offset}"
    return "fts:" + hashlib.sha256(raw.encode()).hexdigest()[:16]

def search_products(query: str, limit: int = 20, offset: int = 0) -> dict:
    cache_key = cache_key_for_search(query, limit, offset)

    # Check cache
    cached = r.get(cache_key)
    if cached:
        return {"results": json.loads(cached), "source": "cache"}

    # Query PostgreSQL
    cursor = conn.cursor()
    cursor.execute("""
        SELECT id, name, description,
               ts_rank(search_vector, plainto_tsquery('english', %s)) AS rank
        FROM products
        WHERE search_vector @@ plainto_tsquery('english', %s)
        ORDER BY rank DESC
        LIMIT %s OFFSET %s
    """, (query, query, limit, offset))

    rows = cursor.fetchall()
    cursor.close()

    results = [
        {"id": row[0], "name": row[1], "description": row[2], "rank": float(row[3])}
        for row in rows
    ]

    # Cache the results
    r.set(cache_key, json.dumps(results), ex=SEARCH_CACHE_TTL)
    return {"results": results, "source": "db"}
```

## Caching Autocomplete Suggestions

```python
AUTOCOMPLETE_TTL = 300

def get_autocomplete(prefix: str, limit: int = 10) -> list:
    cache_key = f"autocomplete:{prefix.lower()}"
    cached = r.get(cache_key)
    if cached:
        return json.loads(cached)

    cursor = conn.cursor()
    cursor.execute("""
        SELECT DISTINCT name FROM products
        WHERE name ILIKE %s
        ORDER BY name
        LIMIT %s
    """, (f"{prefix}%", limit))

    suggestions = [row[0] for row in cursor.fetchall()]
    cursor.close()

    r.set(cache_key, json.dumps(suggestions), ex=AUTOCOMPLETE_TTL)
    return suggestions
```

## Invalidating Search Cache on Product Updates

```python
def invalidate_search_cache():
    """Clear all FTS cache entries when products change."""
    keys = r.keys("fts:*")
    autocomplete_keys = r.keys("autocomplete:*")
    all_keys = keys + autocomplete_keys
    if all_keys:
        r.delete(*all_keys)
        print(f"Invalidated {len(all_keys)} search cache entries")

def update_product(product_id: int, data: dict):
    cursor = conn.cursor()
    cursor.execute(
        "UPDATE products SET name=%s, description=%s WHERE id=%s",
        (data["name"], data["description"], product_id)
    )
    conn.commit()
    cursor.close()
    invalidate_search_cache()
```

## Testing the Cache

```bash
# First call: cache miss, queries PostgreSQL
# Second call: cache hit, served from Redis

redis-cli KEYS "fts:*"
redis-cli TTL fts:abc12345
```

## Summary

Redis caches PostgreSQL full-text search results using a hash of the query as the cache key. Short TTLs (60 seconds) keep results fresh while absorbing repeated identical queries. Invalidate search cache keys when products change to prevent stale results. The same pattern applies to autocomplete prefix queries, which benefit greatly from caching due to their high repetition.

