# How to Use Int8, Int16, Int32, Int64 Data Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Type, Integer, Int8, Int16, Int32, Int64

Description: Learn how to use signed integer types in ClickHouse - Int8, Int16, Int32, and Int64 - including value ranges, storage sizes, and practical query examples.

---

ClickHouse signed integer types allow you to store both positive and negative whole numbers. They are essential when your data can have negative values - temperature readings, financial deltas, elevation data, or offsets. This post covers the four signed integer types, their ranges, storage costs, and how they compare to their unsigned counterparts.

## Overview of Signed Integer Types

Signed integers use one bit for the sign, giving them a symmetric range around zero. Here is a summary:

| Type  | Storage | Min Value              | Max Value             |
|-------|---------|------------------------|-----------------------|
| Int8  | 1 byte  | -128                   | 127                   |
| Int16 | 2 bytes | -32,768                | 32,767                |
| Int32 | 4 bytes | -2,147,483,648         | 2,147,483,647         |
| Int64 | 8 bytes | -9,223,372,036,854,775,808 | 9,223,372,036,854,775,807 |

## Declaring Columns with Signed Integer Types

```sql
CREATE TABLE sensor_readings
(
    reading_id    UInt64,
    sensor_id     UInt32,
    temperature   Int16,   -- Celsius, can be negative
    elevation     Int32,   -- Meters above/below sea level
    delta_bytes   Int64,   -- Change in bytes, can be negative
    signal_level  Int8     -- dBm offset, typically -128 to 0
)
ENGINE = MergeTree()
ORDER BY (sensor_id, reading_id);
```

## Inserting Data

```sql
INSERT INTO sensor_readings
    (reading_id, sensor_id, temperature, elevation, delta_bytes, signal_level) VALUES
(1, 101, -15,  -50,  204800,   -70),
(2, 101, 22,   1200, -512000,  -45),
(3, 102, -40,  -100, 0,        -90);
```

## Comparison with UInt Types

The key difference between signed and unsigned integers is the ability to represent negative values, at the cost of halving the positive range.

```sql
SELECT
    toInt8(-128)   AS int8_min,
    toInt8(127)    AS int8_max,
    toUInt8(0)     AS uint8_min,
    toUInt8(255)   AS uint8_max;
```

Use signed types when:
- Data can naturally be negative (temperatures, coordinate offsets, financial deltas)
- You need to compute differences between unsigned values that might produce negative results

Use unsigned types when:
- Values are always non-negative (IDs, counts, sizes)
- You need the full positive range at a given storage size

## Arithmetic with Signed Integers

```sql
SELECT
    sensor_id,
    avg(temperature)           AS avg_temp,
    min(temperature)           AS min_temp,
    max(temperature)           AS max_temp,
    max(temperature) - min(temperature) AS temp_range
FROM sensor_readings
GROUP BY sensor_id
ORDER BY sensor_id;
```

## Using Int32 for Standard Identifiers

Int32 is the SQL standard `INTEGER` type and maps to what most databases call `INT`. It is suitable for auto-incremented IDs in moderate-scale tables.

```sql
CREATE TABLE products
(
    product_id    Int32,
    price_cents   Int32,   -- Can store price adjustments as negative
    stock_delta   Int16,   -- Change in stock, can be negative (returns)
    weight_grams  Int32
)
ENGINE = MergeTree()
ORDER BY product_id;
```

## Using Int64 for Large-Scale Data

Int64 covers the range needed for most large-scale integer workloads, including Unix timestamps in nanoseconds and very large counters.

```sql
CREATE TABLE financial_transactions
(
    txn_id        UInt64,
    account_id    Int64,
    amount_cents  Int64,   -- Negative for debits, positive for credits
    balance_cents Int64
)
ENGINE = MergeTree()
ORDER BY (account_id, txn_id);

-- Query net flow per account
SELECT
    account_id,
    sum(amount_cents) AS net_flow_cents,
    sum(amount_cents) / 100.0 AS net_flow_dollars
FROM financial_transactions
GROUP BY account_id
ORDER BY net_flow_cents DESC;
```

## Type Casting

```sql
SELECT
    toInt8(-100)    AS as_int8,
    toInt16(-30000) AS as_int16,
    toInt32(-2000000000) AS as_int32,
    CAST('-9000000000000000000', 'Int64') AS as_int64;
```

## Null Handling with Nullable Signed Integers

When a signed integer column might have missing values, wrap it with `Nullable`.

```sql
CREATE TABLE measurements
(
    id          UInt64,
    value       Nullable(Int32),
    corrected   Nullable(Int16)
)
ENGINE = MergeTree()
ORDER BY id;

SELECT
    id,
    ifNull(value, 0) AS value_or_zero
FROM measurements;
```

## Summary

Int8, Int16, Int32, and Int64 are the signed counterparts to ClickHouse's unsigned integer types. Use them whenever your data domain includes negative values - sensor readings, financial deltas, coordinate offsets, or stock changes. Choose the smallest type that fits your range to reduce storage and improve scan performance. For identifiers and counters that are always non-negative, prefer the UInt variants to maximize the available positive range.
