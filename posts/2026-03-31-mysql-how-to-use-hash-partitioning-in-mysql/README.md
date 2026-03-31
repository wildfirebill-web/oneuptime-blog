# How to Use HASH Partitioning in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partitioning, Hash Partitioning, Database Design, Load Distribution

Description: Learn how to implement HASH partitioning in MySQL to evenly distribute rows across partitions using a hash function on a column expression.

---

## What Is HASH Partitioning?

HASH partitioning distributes rows evenly across a fixed number of partitions using a modulo operation on a column expression. Unlike RANGE and LIST partitioning where you define which values go to which partition, HASH partitioning lets MySQL handle the distribution automatically.

Use HASH partitioning when:
- You have no natural grouping of values
- You want even data distribution across partitions
- You want to parallelize reads across partitions

## Basic Syntax

```sql
CREATE TABLE table_name (
  column_definitions
)
PARTITION BY HASH (column_expr)
PARTITIONS N;
```

Where `N` is the number of partitions. MySQL places each row in partition `MOD(column_expr, N)`.

## Example: Partitioning by User ID

```sql
CREATE TABLE user_activity (
  id INT NOT NULL AUTO_INCREMENT,
  user_id INT NOT NULL,
  action VARCHAR(100),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id, user_id)
)
PARTITION BY HASH (user_id)
PARTITIONS 8;
```

## Example: Partitioning by a Function

You can use any integer expression, including functions:

```sql
CREATE TABLE events (
  id INT NOT NULL AUTO_INCREMENT,
  event_date DATE NOT NULL,
  event_type VARCHAR(50),
  payload TEXT,
  PRIMARY KEY (id, event_date)
)
PARTITION BY HASH (TO_DAYS(event_date))
PARTITIONS 4;
```

## How the Distribution Works

For `PARTITIONS 8` and `user_id = 25`:

```text
Partition = MOD(25, 8) = 1  --> goes to partition p1
```

For `user_id = 32`:

```text
Partition = MOD(32, 8) = 0  --> goes to partition p0
```

## Inserting Data

```sql
INSERT INTO user_activity (user_id, action) VALUES
(1, 'login'),
(2, 'view_page'),
(3, 'logout'),
(25, 'purchase');
```

MySQL automatically routes each row to the correct partition.

## Viewing Partition Distribution

```sql
SELECT
  partition_name,
  table_rows
FROM information_schema.partitions
WHERE table_name = 'user_activity'
  AND table_schema = 'mydb'
ORDER BY partition_name;
```

## Querying With Partition Pruning

HASH partitioning supports pruning when filtering on an equality condition for the partition key:

```sql
EXPLAIN SELECT * FROM user_activity WHERE user_id = 25;
```

MySQL computes `MOD(25, 8) = 1` and scans only partition `p1`.

However, range conditions (e.g., `user_id > 10`) will scan all partitions since distribution is not ordered.

## Querying a Specific Partition Directly

```sql
SELECT * FROM user_activity PARTITION (p1);
```

## LINEAR HASH Partitioning

LINEAR HASH uses a linear powers-of-2 algorithm instead of modulo. It is faster for adding or removing partitions but may produce less even distribution:

```sql
CREATE TABLE sessions (
  id INT NOT NULL AUTO_INCREMENT,
  user_id INT NOT NULL,
  created_at DATETIME,
  PRIMARY KEY (id, user_id)
)
PARTITION BY LINEAR HASH (user_id)
PARTITIONS 4;
```

## Adding Partitions

```sql
ALTER TABLE user_activity ADD PARTITION PARTITIONS 4;
```

This increases the total from 8 to 12 partitions. MySQL reorganizes data across all partitions automatically. This can be slow on large tables.

## Coalescing Partitions

To reduce the number of partitions:

```sql
ALTER TABLE user_activity COALESCE PARTITION 2;
```

This reduces the count by 2 (from 8 to 6).

## Choosing the Right Number of Partitions

- Use a power of 2 (4, 8, 16, 32) for LINEAR HASH for best distribution
- Match partition count to the number of CPU cores or storage devices when optimizing parallel I/O
- Avoid too many partitions (hundreds) as MySQL has overhead per partition

## Comparison With Other Partition Types

| Type | Distribution | Use Case |
|------|-------------|---------|
| RANGE | By value ranges | Date/time data, archiving |
| LIST | By discrete values | Categorical data (region, status) |
| HASH | Even distribution via modulo | High-cardinality keys, no natural grouping |
| KEY | Hash on any column type | Non-integer keys |

## Summary

HASH partitioning in MySQL evenly distributes rows across a fixed number of partitions using a modulo operation on a column expression. It is well-suited for high-cardinality integer keys where you want balanced data distribution without defining explicit ranges or value lists. Use LINEAR HASH when you need faster partition management at the cost of slightly uneven distribution.
