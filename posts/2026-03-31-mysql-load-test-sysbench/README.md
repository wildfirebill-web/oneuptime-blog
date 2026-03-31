# How to Load Test MySQL with sysbench

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, sysbench, Load Testing, Performance, Benchmark

Description: Use sysbench to benchmark and load test MySQL with realistic OLTP workloads, measure TPS and latency, and identify performance bottlenecks.

---

## Installing sysbench

```bash
# Ubuntu / Debian
sudo apt-get install -y sysbench

# RHEL / CentOS / Rocky
sudo yum install -y sysbench

# macOS with Homebrew
brew install sysbench

# Verify installation
sysbench --version
```

## Preparing the Test Database

```sql
-- Create a dedicated database for load testing
CREATE DATABASE sbtest;
CREATE USER 'sbtest'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON sbtest.* TO 'sbtest'@'localhost';
FLUSH PRIVILEGES;
```

## Step 1: Prepare Test Data

sysbench provides built-in OLTP Lua scripts. Prepare creates the test tables and inserts rows:

```bash
sysbench \
  --db-driver=mysql \
  --mysql-host=127.0.0.1 \
  --mysql-port=3306 \
  --mysql-user=sbtest \
  --mysql-password=password \
  --mysql-db=sbtest \
  --tables=10 \
  --table-size=1000000 \
  oltp_read_write prepare
```

This creates 10 tables with 1 million rows each.

## Step 2: Run a Read/Write Benchmark

```bash
sysbench \
  --db-driver=mysql \
  --mysql-host=127.0.0.1 \
  --mysql-port=3306 \
  --mysql-user=sbtest \
  --mysql-password=password \
  --mysql-db=sbtest \
  --tables=10 \
  --table-size=1000000 \
  --threads=16 \
  --time=60 \
  --report-interval=10 \
  oltp_read_write run
```

Key output metrics:
- `transactions/sec` (TPS): how many OLTP transactions per second MySQL handles
- `queries/sec` (QPS): raw query throughput
- `95th percentile latency`: latency experienced by the slowest 5% of transactions

## Step 3: Run Read-Only and Write-Only Tests

```bash
# Read-only test (point selects and range scans)
sysbench ... oltp_read_only run

# Write-only test (inserts, updates, deletes)
sysbench ... oltp_write_only run

# Point select only (primary key lookups)
sysbench ... oltp_point_select run
```

## Step 4: Test with Different Thread Counts

Vary `--threads` to find MySQL's saturation point:

```bash
for THREADS in 1 4 8 16 32 64; do
  echo "=== Threads: $THREADS ==="
  sysbench \
    --db-driver=mysql \
    --mysql-host=127.0.0.1 \
    --mysql-user=sbtest \
    --mysql-password=password \
    --mysql-db=sbtest \
    --tables=10 \
    --table-size=1000000 \
    --threads=$THREADS \
    --time=30 \
    oltp_read_write run 2>&1 | grep "transactions:\|95th percentile"
done
```

TPS should increase with thread count until MySQL's thread pool or CPU saturates, after which latency climbs while TPS plateaus.

## Step 5: Compare Before and After Configuration Changes

Use sysbench to measure the effect of tuning changes:

```bash
# Before: record baseline
sysbench ... --time=60 oltp_read_write run > baseline.txt

# Make configuration change, e.g.:
# SET GLOBAL innodb_buffer_pool_size = 4294967296;

# After: run again
sysbench ... --time=60 oltp_read_write run > after_change.txt

# Compare
diff baseline.txt after_change.txt
```

## Step 6: Cleanup

```bash
sysbench \
  --db-driver=mysql \
  --mysql-host=127.0.0.1 \
  --mysql-user=sbtest \
  --mysql-password=password \
  --mysql-db=sbtest \
  --tables=10 \
  oltp_read_write cleanup
```

## Interpreting Results

```
SQL statistics:
    queries performed:
        read:                            2341820
        write:                           669092
        other:                           334546
    transactions:                        167273 (2787.84 per sec.)

Latency (ms):
         min:                                    2.43
         avg:                                    5.73
         max:                                  891.24
         95th percentile:                        8.74
```

A 95th percentile latency under 10 ms with TPS above 2000 is generally healthy for a single-node MySQL instance under OLTP workload on modern hardware.

## Summary

sysbench is the standard tool for MySQL load testing. Use `oltp_read_write` for mixed workloads, vary thread count to find the saturation point, and run tests before and after configuration changes to measure improvement. Always test on a non-production server with representative data sizes to get results that reflect your real workload.
