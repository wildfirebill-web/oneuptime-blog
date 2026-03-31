# How to Create Flat Layout Dictionaries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, Flat Layout, Lookup, Performance

Description: Learn how to create flat layout dictionaries in ClickHouse for fast in-memory key-value lookups with UInt64 numeric keys.

---

## What is a Flat Layout Dictionary

The `flat` layout is the simplest and fastest dictionary layout in ClickHouse. It stores all values in a flat array indexed by the numeric key. Lookups are O(1) array accesses with no hash collisions.

## Constraints

- Key must be a single UInt64 column
- Key values must be in the range [0, 500,000] by default
- Best for small, dense numeric key spaces (e.g., status codes, category IDs)

## Creating a Flat Dictionary from a ClickHouse Table

```sql
-- Source table
CREATE TABLE status_codes_data (
    id UInt64,
    name String,
    description String
) ENGINE = MergeTree() ORDER BY id;

INSERT INTO status_codes_data VALUES
(1, 'active', 'Account is active'),
(2, 'suspended', 'Account is suspended'),
(3, 'deleted', 'Account has been deleted'),
(4, 'pending', 'Account pending verification');

-- Create flat dictionary
CREATE DICTIONARY status_codes_dict (
    id UInt64,
    name String DEFAULT 'unknown',
    description String DEFAULT ''
)
PRIMARY KEY id
SOURCE(CLICKHOUSE(TABLE 'status_codes_data' DB 'default'))
LAYOUT(FLAT())
LIFETIME(MIN 300 MAX 600);
```

## Create from a CSV File

```sql
CREATE DICTIONARY product_categories (
    id UInt64,
    category_name String DEFAULT 'uncategorized',
    parent_id UInt64 DEFAULT 0
)
PRIMARY KEY id
SOURCE(FILE(PATH '/var/lib/clickhouse/user_files/categories.csv' FORMAT CSV))
LAYOUT(FLAT())
LIFETIME(3600);
```

## Querying with dictGet

```sql
SELECT
    order_id,
    status_code,
    dictGetString('status_codes_dict', 'name', status_code) AS status_name,
    dictGetString('status_codes_dict', 'description', status_code) AS status_desc
FROM orders
LIMIT 10;
```

## Check if Key Exists

```sql
SELECT
    user_id,
    status_code,
    dictHas('status_codes_dict', status_code) AS is_valid_status
FROM users
WHERE NOT dictHas('status_codes_dict', status_code);
```

## Monitor Dictionary Status

```sql
SELECT
    name,
    status,
    element_count,
    bytes_allocated,
    formatReadableSize(bytes_allocated) AS size,
    last_successful_update_time
FROM system.dictionaries
WHERE name = 'status_codes_dict';
```

## Reload a Dictionary

```sql
SYSTEM RELOAD DICTIONARY status_codes_dict;
```

## Configure Max Array Size

For keys up to 1,000,000, increase the array limit:

```sql
LAYOUT(FLAT(INITIAL_ARRAY_SIZE 1000000 MAX_ARRAY_SIZE 1000000))
```

## Performance Comparison

Flat layout lookups are consistently the fastest:

```text
Layout       | Lookup speed     | Memory usage
flat         | O(1) array       | High (pre-allocated)
hashed       | O(1) hash table  | Medium
cache        | O(1) cache hit   | Configurable
```

## Summary

Flat layout dictionaries in ClickHouse offer the fastest possible lookup performance for small integer key spaces. They are ideal for translating compact numeric codes (status codes, category IDs, country codes) into human-readable strings at query time with minimal overhead.
