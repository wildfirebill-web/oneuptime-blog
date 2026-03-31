# How to Use RedisInsight for Key Browsing and Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RedisInsight, Key Management, Browser, GUI

Description: Learn how to browse, filter, inspect, and manage Redis keys using the RedisInsight browser with support for all data types and TTL management.

---

RedisInsight's Browser view is a visual alternative to `redis-cli KEYS` and `SCAN`. It lets you explore your keyspace safely, view values, edit TTLs, and delete keys without writing commands.

## Opening the Browser

After connecting a database in RedisInsight, click "Browser" in the left sidebar. The browser loads a paginated list of keys using `SCAN` under the hood, which is safe for production use.

## Filtering Keys by Pattern

Use the search bar to filter keys by glob pattern:

```text
user:*          -> all keys starting with user:
session:*:data  -> session keys with a :data suffix
*:error         -> any key ending in :error
```

You can also filter by data type using the type dropdown: String, Hash, List, Set, Sorted Set, Stream, JSON, etc.

## Viewing Key Details

Click any key to open its detail panel on the right. For each type:

- String: displays the raw value with encoding info
- Hash: shows a table of field-value pairs
- List: shows indexed values with scroll
- Set: shows all members
- Sorted Set: shows members with scores, sortable
- Stream: shows entries with field-value messages

## Editing Values

For most types, click the edit icon next to a value to modify it inline. For hashes, click a field value to update it:

```text
1. Click the hash key "user:123"
2. Find field "email"
3. Click the pencil icon
4. Type the new email
5. Click "Apply"
```

RedisInsight sends the corresponding `HSET` command to Redis.

## Adding New Keys

Click the "+" button in the Browser toolbar to add a new key:

```text
Key name: product:9999
Type: Hash
Fields:
  name: Widget Pro
  price: 49.99
  stock: 100
TTL: 3600
```

## Managing TTL

To view or update a key's TTL:

1. Click the key
2. The detail panel shows "TTL: 1800s" (or "No Expiry")
3. Click the edit icon next to TTL
4. Enter a new TTL in seconds
5. Click "Apply"

To remove expiration, set TTL to `-1`.

## Bulk Deleting Keys

Use the checkbox on the left of each row to select multiple keys, then click the trash icon:

```text
1. Search for pattern: temp:*
2. Select all (checkbox in header)
3. Click Delete
4. Confirm deletion
```

This triggers individual `DEL` commands per key, so use with care on large key sets.

## Refreshing the Browser

The browser does not auto-refresh. Click the refresh icon or press `Ctrl+R` to reload the key list. This is intentional to prevent accidental overloading of production instances.

## Viewing Key Memory Usage

Hover over a key and look for the memory badge. RedisInsight calls `MEMORY USAGE key` in the background to show bytes used per key. This helps identify unexpectedly large keys.

## Summary

RedisInsight's Browser provides a safe, pattern-based way to explore your keyspace, view and edit all data types, manage TTLs, and delete keys without using the CLI. It uses `SCAN` internally to avoid blocking the server during key enumeration.
