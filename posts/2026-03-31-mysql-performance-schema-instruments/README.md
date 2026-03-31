# How to Configure Performance Schema Instruments in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Instrument, Monitoring, Database

Description: Learn how to enable, disable, and configure Performance Schema instruments in MySQL to control exactly what the server monitors and measures.

---

MySQL's Performance Schema collects detailed runtime statistics about server execution. Instruments are the hooks placed throughout the MySQL server code that define what gets measured. By configuring instruments, you control the granularity and overhead of monitoring without disabling the Performance Schema entirely.

## What Are Performance Schema Instruments

Instruments follow a naming hierarchy using forward-slash delimiters. For example:

- `wait/io/file/sql/FRM` - file I/O waits for FRM files
- `statement/sql/select` - SELECT statement execution
- `memory/sql/thd::main_mem_root` - memory allocation events
- `stage/sql/Sorting result` - query execution stages

Each instrument has an `ENABLED` flag (whether events are collected) and a `TIMED` flag (whether timing data is captured).

## Viewing Instruments

To see all available instruments and their current state:

```sql
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
LIMIT 20;
```

To filter by category:

```sql
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%'
ORDER BY NAME;
```

To count instruments by category:

```sql
SELECT SUBSTRING_INDEX(NAME, '/', 2) AS category,
       COUNT(*) AS total,
       SUM(ENABLED = 'YES') AS enabled_count
FROM performance_schema.setup_instruments
GROUP BY category
ORDER BY total DESC;
```

## Enabling and Disabling Instruments

To enable all statement instruments:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';
```

To enable all wait instruments:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/%';
```

To disable memory instruments to reduce overhead:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO'
WHERE NAME LIKE 'memory/%';
```

To enable a specific instrument:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME = 'wait/io/table/sql/handler';
```

## Enabling Instruments Permanently in my.cnf

Runtime changes to `setup_instruments` are reset on server restart. To make them permanent, add entries to `my.cnf`:

```bash
[mysqld]
performance-schema-instrument='statement/%=ON'
performance-schema-instrument='wait/io/%=ON'
performance-schema-instrument='memory/%=OFF'
```

Then restart MySQL:

```bash
sudo systemctl restart mysqld
```

Verify the settings took effect:

```sql
SELECT NAME, ENABLED, TIMED
FROM performance_schema.setup_instruments
WHERE NAME LIKE 'statement/%'
LIMIT 5;
```

## Common Instrument Categories

| Category | Purpose | Overhead |
|---|---|---|
| `statement/%` | SQL statement tracking | Low-Medium |
| `wait/io/file/%` | File I/O operations | Medium |
| `wait/io/table/%` | Table I/O operations | Medium-High |
| `wait/lock/%` | Lock wait tracking | Low |
| `stage/%` | Query stage tracking | Medium |
| `memory/%` | Memory allocation | High |
| `transaction` | Transaction events | Low |

## Balancing Overhead and Coverage

Enabling all instruments creates overhead. A practical approach for production:

```sql
-- Disable everything first
UPDATE performance_schema.setup_instruments
SET ENABLED = 'NO', TIMED = 'NO';

-- Enable only statements and locks
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%'
   OR NAME LIKE 'wait/lock/%'
   OR NAME LIKE 'transaction%';
```

For investigating I/O bottlenecks, temporarily enable file and table I/O instruments, collect data, then disable them again.

## Summary

Performance Schema instruments are granular controls that determine what MySQL measures at runtime. You enable or disable them via the `setup_instruments` table either at runtime with `UPDATE` or permanently via `my.cnf` configuration. Focus on statement and lock instruments for everyday monitoring, and enable I/O or memory instruments temporarily when diagnosing specific performance problems.
