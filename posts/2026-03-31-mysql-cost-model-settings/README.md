# How to Configure MySQL Cost Model Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Optimizer, Cost Model, Performance, Configuration

Description: Learn how MySQL's cost model works, how to view and customize cost model settings, and how adjusting costs influences the query optimizer's execution plan choices.

---

## Overview

MySQL's query optimizer estimates the cost of different execution plans and chooses the lowest-cost option. The cost model consists of configurable cost constants that represent the relative expense of operations like disk reads, memory reads, and row comparisons. Tuning these constants lets you guide the optimizer when its default estimates lead to suboptimal plans.

## Understanding the Cost Model Tables

MySQL stores cost model settings in two tables in the `mysql` database:

```sql
-- Server-level operation costs
SELECT * FROM mysql.server_cost;

-- Engine-level operation costs (per storage engine)
SELECT * FROM mysql.engine_cost;
```

## Server Cost Constants

```sql
SELECT cost_name, default_value, cost_value, last_update, comment
FROM mysql.server_cost
ORDER BY cost_name;
```

Key constants and their defaults:

```text
cost_name                          | default
-----------------------------------+--------
disk_temptable_create_cost         | 20.0
disk_temptable_row_cost            | 0.5
key_compare_cost                   | 0.05
memory_temptable_create_cost       | 1.0
memory_temptable_row_cost          | 0.1
row_evaluate_cost                  | 0.1
```

## Engine Cost Constants

```sql
SELECT engine_name, cost_name, default_value, cost_value
FROM mysql.engine_cost
ORDER BY engine_name, cost_name;
```

Key constants:

```text
engine_name | cost_name                    | default
------------+------------------------------+--------
default     | io_block_read_cost           | 1.0
default     | memory_block_read_cost       | 0.25
```

## Modifying Cost Constants

Increase the relative cost of disk reads to discourage full table scans on large tables:

```sql
-- Make disk reads appear more expensive (discourages full table scans)
UPDATE mysql.engine_cost
SET cost_value = 2.0
WHERE cost_name = 'io_block_read_cost';

-- Make in-memory operations cheaper (favors index scans)
UPDATE mysql.engine_cost
SET cost_value = 0.1
WHERE cost_name = 'memory_block_read_cost';

-- Reload the cost model for changes to take effect
FLUSH OPTIMIZER_COSTS;
```

## Resetting to Defaults

```sql
-- Reset all server costs to defaults
UPDATE mysql.server_cost SET cost_value = NULL;
UPDATE mysql.engine_cost SET cost_value = NULL;

FLUSH OPTIMIZER_COSTS;
```

When `cost_value` is NULL, MySQL uses `default_value`.

## Verifying the Impact with EXPLAIN

Use EXPLAIN to see how cost changes affect execution plans:

```sql
-- Show query cost before adjustment
EXPLAIN FORMAT=JSON
SELECT o.id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2024-01-01';
```

Look for the `query_cost` field in the JSON output:

```json
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "142.35"
    }
  }
}
```

## Engine-Specific Cost Overrides

You can set different costs for specific storage engines:

```sql
-- Set InnoDB-specific disk read cost (overrides 'default')
INSERT INTO mysql.engine_cost
    (engine_name, device_type, cost_name, cost_value, comment)
VALUES
    ('InnoDB', 0, 'io_block_read_cost', 1.5, 'InnoDB SSD adjustment')
ON DUPLICATE KEY UPDATE
    cost_value = 1.5;

FLUSH OPTIMIZER_COSTS;
```

## When to Adjust Cost Constants

```text
Increase io_block_read_cost when:
  - Tables are on slow spinning disks
  - You want to push the optimizer toward more index usage

Decrease io_block_read_cost when:
  - Storage is NVMe/SSD and disk reads are nearly as fast as memory
  - The optimizer over-estimates scan costs and avoids correct full scans

Adjust memory_block_read_cost when:
  - You have a large buffer pool and most data is cached in memory
  - Lower value makes cached range scans more attractive
```

## Summary

MySQL's cost model tables (`mysql.server_cost` and `mysql.engine_cost`) store the constants the optimizer uses when estimating execution plan costs. Adjust `io_block_read_cost` higher on slow storage to discourage unnecessary full table scans, or lower on SSD/NVMe to make full scans more competitive. Always call `FLUSH OPTIMIZER_COSTS` after making changes, verify the impact with `EXPLAIN FORMAT=JSON`, and reset to defaults with NULL values if adjustments cause regressions.
