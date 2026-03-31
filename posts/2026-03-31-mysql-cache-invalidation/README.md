# How to Invalidate Cache After MySQL Data Changes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Cache, Invalidation

Description: Learn strategies for invalidating cache entries after MySQL data changes, including key-based deletion, tag-based invalidation, and CDC-driven cache purging.

---

Cache invalidation is the process of removing or updating cached data when the underlying MySQL data changes. Getting it wrong leads to stale reads (users seeing outdated data) or over-invalidation (unnecessary cache misses). There is no universally correct strategy - the right approach depends on how your data is structured and how frequently it changes.

## Strategy 1: Direct Key Deletion

The simplest strategy: delete the cache key immediately after the MySQL write:

```python
def update_user_email(user_id, new_email):
    cursor = db.cursor()
    cursor.execute('UPDATE users SET email=%s WHERE id=%s', (new_email, user_id))
    db.commit()

    # Invalidate single record cache
    r.delete(f'user:{user_id}')
```

This is reliable and easy to reason about. The downside is that every write operation must know all the cache keys it affects.

## Strategy 2: Tag-Based Invalidation

Group related cache keys under a shared tag so you can invalidate them all at once:

```python
def cache_set_with_tag(key, value, tag, ttl=300):
    r.setex(key, ttl, value)
    r.sadd(f'tag:{tag}', key)
    r.expire(f'tag:{tag}', ttl + 60)


def invalidate_tag(tag):
    keys = r.smembers(f'tag:{tag}')
    if keys:
        r.delete(*keys)
    r.delete(f'tag:{tag}')


# Cache product and product list under the same tag
def cache_product(product):
    tag = f'product-category:{product["category_id"]}'
    cache_set_with_tag(f'product:{product["id"]}', json.dumps(product), tag)
    cache_set_with_tag(f'products:cat:{product["category_id"]}', json.dumps([product]), tag)


def update_product(product_id, data):
    cursor = db.cursor(dictionary=True)
    cursor.execute('UPDATE products SET name=%s, price=%s WHERE id=%s',
                   (data['name'], data['price'], product_id))
    db.commit()

    # Get category to invalidate tag
    cursor.execute('SELECT category_id FROM products WHERE id=%s', (product_id,))
    row = cursor.fetchone()
    if row:
        invalidate_tag(f'product-category:{row["category_id"]}')
```

## Strategy 3: Pattern-Based Deletion

Delete all keys matching a prefix pattern (use cautiously on large Redis instances):

```python
def invalidate_user_cache(user_id):
    pattern = f'user:{user_id}:*'
    keys = r.keys(pattern)
    if keys:
        r.delete(*keys)
```

`r.keys()` scans all keys and can block Redis. For production use, prefer `r.scan_iter()`:

```python
def invalidate_by_pattern(pattern):
    keys = list(r.scan_iter(pattern))
    if keys:
        r.delete(*keys)
```

## Strategy 4: CDC-Driven Invalidation with Debezium

Change Data Capture (CDC) listens to the MySQL binary log and emits an event for every row change. A consumer reads these events and invalidates the affected cache keys:

```python
# Kafka consumer for Debezium MySQL CDC events
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'myapp.myapp.products',
    bootstrap_servers='localhost:9092',
    value_deserializer=lambda v: json.loads(v)
)

for message in consumer:
    payload = message.value.get('payload', {})
    op = payload.get('op')  # c=create, u=update, d=delete

    if op in ('u', 'd') and payload.get('before'):
        product_id = payload['before']['id']
        r.delete(f'product:{product_id}')
        print(f"Invalidated cache for product:{product_id} (op={op})")
```

CDC decouples invalidation from the application layer, ensuring cache consistency even when MySQL is updated by other services or direct SQL.

## Using MySQL Triggers for Invalidation Signals

Create triggers that insert invalidation records into a table, which your application polls:

```sql
CREATE TABLE cache_invalidation_queue (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  entity     VARCHAR(50) NOT NULL,
  entity_id  INT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TRIGGER after_product_update
AFTER UPDATE ON products
FOR EACH ROW
INSERT INTO cache_invalidation_queue (entity, entity_id) VALUES ('product', NEW.id);
```

## Summary

Cache invalidation strategies range from simple direct key deletion to sophisticated CDC-driven approaches. Direct deletion is easiest to implement. Tag-based invalidation handles related key groups. CDC-driven invalidation via Debezium provides automatic consistency for multi-service environments. Choose based on your consistency requirements and infrastructure complexity.
