# How to Use Enum8 and Enum16 Data Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Types, Enum, Enum8, Enum16

Description: Learn how to use Enum8 and Enum16 in ClickHouse for storage-efficient categorical data, including defining values, querying, and altering enum definitions.

---

When a column can only take a small set of known string values - HTTP methods, order statuses, log levels, event types - storing them as raw strings wastes space and slows down GROUP BY queries. ClickHouse's Enum types map string labels to integer codes internally, giving you the readability of string values with the storage efficiency and comparison speed of integers. This post covers Enum8 and Enum16, how to define and use them, and how to manage their definitions as your schema evolves.

## Enum8 vs Enum16

| Type   | Underlying Storage | Max Distinct Values | Storage per Row |
|--------|--------------------|---------------------|-----------------|
| Enum8  | Int8               | 256                 | 1 byte          |
| Enum16 | Int16              | 65,536              | 2 bytes         |

`Enum8` is sufficient for the vast majority of categorical columns. `Enum16` is needed when the number of distinct labels exceeds 256.

## Defining an Enum Column

Enum values are defined inline in the column definition as `'label' = integer_code` pairs.

```sql
CREATE TABLE orders
(
    order_id     UInt64,
    customer_id  UInt64,
    status       Enum8('pending' = 1, 'processing' = 2, 'shipped' = 3,
                        'delivered' = 4, 'cancelled' = 5, 'refunded' = 6),
    priority     Enum8('low' = 1, 'normal' = 2, 'high' = 3, 'urgent' = 4),
    created_at   DateTime
)
ENGINE = MergeTree()
ORDER BY (customer_id, order_id);
```

## Inserting Enum Values

Insert using the string label. ClickHouse stores the corresponding integer internally.

```sql
INSERT INTO orders
    (order_id, customer_id, status, priority, created_at) VALUES
(1001, 501, 'pending',    'normal', '2026-03-31 09:00:00'),
(1002, 502, 'shipped',    'high',   '2026-03-31 09:15:00'),
(1003, 501, 'delivered',  'low',    '2026-03-31 09:30:00'),
(1004, 503, 'cancelled',  'urgent', '2026-03-31 09:45:00');
```

You can also insert by integer code:

```sql
INSERT INTO orders (order_id, customer_id, status, priority, created_at) VALUES
(1005, 504, 2, 2, '2026-03-31 10:00:00');   -- 2 = 'processing', 2 = 'normal'
```

## Querying Enum Columns

Enum columns behave like strings in SELECT and WHERE clauses.

```sql
-- Filter by enum label
SELECT order_id, customer_id, status, priority
FROM orders
WHERE status IN ('pending', 'processing')
  AND priority = 'high';

-- Aggregate by enum value
SELECT
    status,
    count()   AS order_count
FROM orders
GROUP BY status
ORDER BY order_count DESC;
```

## Enum in ORDER BY

Enums sort by their integer code, not alphabetically. This means the sort order is determined by how you defined the codes.

```sql
-- Orders sort by the integer values: 1=pending, 2=processing, ...
SELECT order_id, status
FROM orders
ORDER BY status ASC;

-- To sort alphabetically, cast to String first
SELECT order_id, status
FROM orders
ORDER BY CAST(status, 'String') ASC;
```

## Using Enum16 for Larger Value Sets

Use Enum16 when you have more than 256 distinct categorical values.

```sql
CREATE TABLE product_catalog
(
    product_id   UInt64,
    category     Enum16(
        'electronics' = 1,
        'clothing'    = 2,
        'books'       = 3,
        'home'        = 4,
        'sports'      = 5,
        'toys'        = 6,
        'food'        = 7,
        'automotive'  = 8,
        'garden'      = 9,
        'health'      = 10
    ),
    subcategory  Enum16('laptops' = 1, 'phones' = 2, 'tablets' = 3,
                         'cameras' = 4, 'audio' = 5),
    in_stock     Enum8('yes' = 1, 'no' = 2, 'preorder' = 3)
)
ENGINE = MergeTree()
ORDER BY product_id;
```

## Adding New Enum Values with ALTER TABLE

You can extend an Enum type by adding new values using `ALTER TABLE MODIFY COLUMN`. Existing data remains valid because existing codes are unchanged.

```sql
-- Add a new status value to an existing Enum8 column
ALTER TABLE orders
MODIFY COLUMN status Enum8(
    'pending'    = 1,
    'processing' = 2,
    'shipped'    = 3,
    'delivered'  = 4,
    'cancelled'  = 5,
    'refunded'   = 6,
    'on_hold'    = 7   -- New value added
);
```

Note: You cannot remove values that are in use, and you cannot change existing numeric codes without rewriting the data.

## Converting Enum to String and Integer

```sql
SELECT
    order_id,
    status,
    CAST(status, 'String')  AS status_string,
    CAST(status, 'Int8')    AS status_int
FROM orders;
```

## Storage Efficiency Comparison

```sql
-- Demonstrating storage savings: Enum8 vs String for status
CREATE TABLE status_comparison
(
    id             UInt32,
    status_enum    Enum8('pending' = 1, 'shipped' = 2, 'delivered' = 3),
    status_string  String
)
ENGINE = Memory;

INSERT INTO status_comparison VALUES
(1, 'pending',   'pending'),
(2, 'shipped',   'shipped'),
(3, 'delivered', 'delivered');

-- The enum column stores 1 byte per row instead of 7-9 bytes
SELECT
    id,
    status_enum,
    status_string,
    toTypeName(status_enum)   AS enum_type,
    toTypeName(status_string) AS string_type
FROM status_comparison;
```

## Summary

Enum8 and Enum16 are ClickHouse's categorical types, storing string labels as compact integers - 1 byte for Enum8 (up to 256 values) and 2 bytes for Enum16 (up to 65,536 values). They provide the readability of string comparisons in queries while delivering the storage and GROUP BY performance of integers. Use Enum8 for columns like status, log level, or HTTP method, and Enum16 when the category space is larger. Extend enum definitions safely with `ALTER TABLE MODIFY COLUMN` as your domain grows.
