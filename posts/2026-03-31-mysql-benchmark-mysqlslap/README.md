# How to Benchmark MySQL Performance with mysqlslap

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Benchmark, Performance, Mysqlslap, Testing

Description: Learn how to use the built-in mysqlslap tool to benchmark MySQL by simulating concurrent client connections and measuring query execution times.

---

## Introduction

mysqlslap is a diagnostic program bundled with MySQL that emulates multiple clients accessing a database simultaneously. Unlike external tools, mysqlslap requires no additional installation and is ideal for quick performance checks after schema or configuration changes.

## Basic Usage

Connect to your MySQL server and run a simple auto-generated workload:

```bash
mysqlslap \
  --user=root \
  --password \
  --auto-generate-sql \
  --concurrency=50 \
  --iterations=3
```

`--concurrency` sets the number of simultaneous clients, and `--iterations` runs the test multiple times to average results.

## Customizing the Schema

You can provide your own schema and queries instead of using the auto-generated ones:

```bash
mysqlslap \
  --user=root \
  --password \
  --create="CREATE TABLE test.bench (id INT PRIMARY KEY AUTO_INCREMENT, data VARCHAR(255));" \
  --query="INSERT INTO test.bench (data) VALUES (REPEAT('x', 200)); SELECT * FROM test.bench LIMIT 10;" \
  --concurrency=100 \
  --iterations=5 \
  --delimiter=";"
```

## Testing with a SQL File

Store your benchmark queries in a file for reuse:

```sql
SELECT id, data FROM test.bench WHERE id = FLOOR(1 + RAND() * 10000);
UPDATE test.bench SET data = REPEAT('y', 100) WHERE id = FLOOR(1 + RAND() * 10000);
```

Then run:

```bash
mysqlslap \
  --user=root \
  --password \
  --query=/tmp/bench_queries.sql \
  --create-schema=bench_db \
  --concurrency=25,50,100 \
  --iterations=3
```

Passing multiple concurrency values (25,50,100) causes mysqlslap to run each load level sequentially so you can measure how performance scales.

## Sample Output

```text
Benchmark
    Average number of seconds to run all queries: 1.242 seconds
    Minimum number of seconds to run all queries: 1.187 seconds
    Maximum number of seconds to run all queries: 1.301 seconds
    Number of clients running queries: 50
    Average number of queries per client: 0
```

## Auto-Generate Options

mysqlslap supports several flags to control auto-generated workloads:

```bash
mysqlslap \
  --user=root \
  --password \
  --auto-generate-sql \
  --auto-generate-sql-load-type=mixed \
  --auto-generate-sql-write-number=1000 \
  --number-of-queries=5000 \
  --concurrency=20 \
  --iterations=3
```

- `--auto-generate-sql-load-type` accepts `read`, `write`, `update`, `key`, or `mixed`
- `--auto-generate-sql-write-number` sets how many rows to pre-populate
- `--number-of-queries` limits total queries per client run

## Interpreting Results

The key metric from mysqlslap is the average seconds to complete all queries. Compare values across concurrency levels to identify when contention causes latency to spike. A linear increase suggests good scalability; an exponential jump indicates lock contention or resource saturation.

## Summary

mysqlslap is a quick and convenient tool for MySQL benchmarking without any external dependencies. It simulates realistic concurrent client loads using either custom SQL files or auto-generated queries. Use it after tuning InnoDB settings, adding indexes, or changing query structures to quantify the impact of your changes.
