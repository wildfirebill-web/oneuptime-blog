# How to Use FT.ALIASDEL in Redis to Remove Index Aliases

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RediSearch, Search

Description: Learn how to use FT.ALIASDEL in Redis to remove a search index alias, cleaning up stale alias references after index migrations.

---

`FT.ALIASDEL` removes an existing alias from a RediSearch index. This is the cleanup step after an index migration or when an alias is no longer needed. After deletion, any query using that alias will fail until the alias is re-created or the application is updated to use the index name directly.

## Prerequisites

Ensure RediSearch is available:

```bash
redis-cli MODULE LIST
```

You should see `ft` or `search` in the list.

## Syntax

```text
FT.ALIASDEL <alias>
```

- `alias`: the name of the alias to remove

## Basic Example

Create an index and alias, then remove the alias:

```bash
# Create an index
FT.CREATE products_v1
  ON HASH
  PREFIX 1 product:
  SCHEMA name TEXT price NUMERIC

# Add an alias
FT.ALIASADD products products_v1

# Confirm the alias works
FT.SEARCH products "*"

# Remove the alias
FT.ALIASDEL products
```

After running `FT.ALIASDEL products`, any search against `products` will return:

```text
(error) ERR Unknown index name (or name is an alias to an erased index)
```

## When to Use FT.ALIASDEL

**After completing an index migration:** Once both old and new aliases are cleaned up and the application points directly to the new index name.

**When retiring a feature:** Remove aliases tied to deprecated features to avoid confusion.

**Before reassigning:** If you want to recreate an alias pointing to a different index (rather than using `FT.ALIASUPDATE`):

```bash
FT.ALIASDEL my_alias
FT.ALIASADD my_alias new_index
```

## Python Example

```python
import redis

r = redis.Redis(host="127.0.0.1", port=6379, decode_responses=True)

# Remove alias
try:
    r.execute_command("FT.ALIASDEL", "products")
    print("Alias 'products' removed successfully")
except redis.ResponseError as e:
    if "Alias does not exist" in str(e):
        print("Alias does not exist - nothing to remove")
    else:
        raise
```

## Error Cases

If the alias does not exist:

```text
(error) ERR Alias does not exist
```

Handle this gracefully in automation scripts by catching the error and treating it as a no-op.

## Complete Alias Lifecycle Example

```bash
# Phase 1: Create index and alias
FT.CREATE idx_v1 ON HASH PREFIX 1 doc: SCHEMA title TEXT
FT.ALIASADD docs idx_v1

# Phase 2: Rebuild index with new schema
FT.CREATE idx_v2 ON HASH PREFIX 1 doc: SCHEMA title TEXT body TEXT

# Phase 3: Swap alias atomically
FT.ALIASUPDATE docs idx_v2

# Phase 4: Drop old index
FT.DROPINDEX idx_v1

# Phase 5: If alias is no longer needed, remove it
FT.ALIASDEL docs
```

## Summary

`FT.ALIASDEL` is the simple but essential counterpart to `FT.ALIASADD`. It removes aliases that are no longer needed, preventing stale references from causing confusion. Use it as part of index migration cleanup workflows, and handle the "alias does not exist" error gracefully in scripts to make the operation idempotent.
