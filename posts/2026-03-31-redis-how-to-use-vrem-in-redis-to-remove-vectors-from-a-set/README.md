# How to Use VREM in Redis to Remove Vectors from a Set

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Vector Set, Embedding, Data Management

Description: Learn how to use VREM in Redis to remove specific vector elements from a Vector Set, keeping your index fresh and accurate.

---

## What Is VREM?

`VREM` removes one or more elements (and their associated vector embeddings) from a Redis Vector Set. This is essential for keeping your vector index synchronized with your data - for example, when a product is discontinued, a user account is deleted, or a document is unpublished.

## Syntax

```text
VREM key element [element ...]
```

Returns the number of elements successfully removed.

## Basic Usage

```bash
# First, add some vectors
VADD products prod:1001 VALUES 4 0.1 0.2 0.3 0.4
VADD products prod:1002 VALUES 4 0.5 0.6 0.7 0.8
VADD products prod:1003 VALUES 4 0.2 0.3 0.4 0.5

# Verify count
VCARD products
# Returns: 3

# Remove a single element
VREM products prod:1002
# Returns: (integer) 1

# Verify count after removal
VCARD products
# Returns: 2
```

## Removing Multiple Elements

```bash
# Remove multiple elements in one call
VREM products prod:1001 prod:1003
# Returns: (integer) 2  (both removed)

# Check count
VCARD products
# Returns: 0
```

## Handling Non-Existent Elements

If an element does not exist, VREM ignores it and only counts the ones that were actually removed:

```bash
VADD products prod:1001 VALUES 4 0.1 0.2 0.3 0.4

# Remove mix of existing and non-existing
VREM products prod:1001 prod:9999
# Returns: (integer) 1  (only prod:1001 was removed)
```

## Python Example: Keeping Index in Sync

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def remove_from_index(key: str, element_ids: list) -> int:
    """Remove elements from the vector index. Returns count removed."""
    if not element_ids:
        return 0
    result = r.execute_command("VREM", key, *element_ids)
    return int(result)

def sync_deletions(vector_key: str, deleted_ids: list):
    """Sync deletions from primary database to vector index."""
    removed = remove_from_index(vector_key, deleted_ids)
    print(f"Removed {removed}/{len(deleted_ids)} vectors from index '{vector_key}'")

# Simulate product deletions
r.execute_command("VADD", "products:vectors", "prod:1001", "VALUES", "4", "0.1", "0.2", "0.3", "0.4")
r.execute_command("VADD", "products:vectors", "prod:1002", "VALUES", "4", "0.5", "0.6", "0.7", "0.8")
r.execute_command("VADD", "products:vectors", "prod:1003", "VALUES", "4", "0.9", "0.8", "0.7", "0.6")

print(f"Before: {r.execute_command('VCARD', 'products:vectors')} vectors")

# Products deleted from the database
deleted_products = ["prod:1001", "prod:1002"]
sync_deletions("products:vectors", deleted_products)

print(f"After: {r.execute_command('VCARD', 'products:vectors')} vectors")
```

## Soft Delete Pattern

In some systems, you may want a soft delete before permanent removal:

```python
def soft_delete_vector(key: str, element_id: str, graveyard_key: str):
    """
    Move a vector to a 'deleted' set before removing from active index.
    This allows rollback if needed.
    """
    # Record deletion in a regular Redis set for auditing
    r.sadd(graveyard_key, element_id)

    # Remove from active vector index
    removed = r.execute_command("VREM", key, element_id)
    return removed > 0

def restore_from_soft_delete(key: str, element_id: str, embedding: list, graveyard_key: str):
    """Restore a soft-deleted element back to the vector index."""
    if not r.sismember(graveyard_key, element_id):
        return False

    dim = len(embedding)
    cmd = ["VADD", key, element_id, "VALUES", str(dim)] + [str(v) for v in embedding]
    r.execute_command(*cmd)
    r.srem(graveyard_key, element_id)
    return True
```

## Batch Deletion with Pipeline

For large-scale removals, use Redis pipelines:

```python
def batch_remove_vectors(key: str, element_ids: list, batch_size: int = 500) -> int:
    """Remove many elements efficiently using VREM with batching."""
    total_removed = 0
    for i in range(0, len(element_ids), batch_size):
        batch = element_ids[i:i + batch_size]
        removed = r.execute_command("VREM", key, *batch)
        total_removed += int(removed)
    return total_removed

# Remove 1000 stale vectors
stale_ids = [f"doc:{i}" for i in range(1000)]
count = batch_remove_vectors("docs:vectors", stale_ids, batch_size=100)
print(f"Removed {count} stale vectors")
```

## Checking Before Removal

If you need to confirm an element exists before removing it:

```bash
# Use VSIM to check if element exists by querying it
# Or use a separate existence tracking set
SISMEMBER product_ids "prod:1001"
# Returns: 1 (exists)

VREM products prod:1001
```

```python
def safe_remove(key: str, element_id: str) -> bool:
    """Remove an element and return True if it was found and removed."""
    result = r.execute_command("VREM", key, element_id)
    return int(result) > 0
```

## Summary

`VREM` removes named vector elements from a Redis Vector Set, returning the count of successfully removed items. It handles non-existent elements gracefully, ignoring them without error. Use it to keep your vector index synchronized with your primary data store - removing discontinued products, deleted documents, or deactivated user profiles. For large-scale removals, batch multiple element names into a single VREM call or use iteration with batching for thousands of deletions.
