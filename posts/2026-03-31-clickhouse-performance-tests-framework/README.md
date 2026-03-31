# How to Use ClickHouse Performance Tests Framework

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Performance, Testing, Framework, Benchmark, Regression

Description: Use the ClickHouse performance tests framework to run standardized benchmarks and detect performance regressions across versions.

---

The ClickHouse project ships a built-in performance testing framework located in `tests/performance`. This framework is used by the ClickHouse team to catch query regressions between releases and can be adapted for your own workloads.

## Overview of the Framework

The performance tests framework consists of XML test definition files and a Python runner (`perf.py`). Each test file defines:
- Preconditions (tables to create, data to load)
- Queries to benchmark
- Stop conditions (iterations, time limit)

## Running a Single Performance Test

```bash
# Clone ClickHouse repository
git clone --depth=1 https://github.com/ClickHouse/ClickHouse.git
cd ClickHouse

# Install dependencies
pip3 install -r tests/performance/requirements.txt

# Run a specific test
python3 tests/performance/perf.py \
  --host localhost --port 9000 \
  --test tests/performance/hits.xml
```

## Writing a Custom Test File

Create an XML file defining your benchmark:

```xml
<test>
    <name>my_events_benchmark</name>
    <preconditions>
        <create_query>
            CREATE TABLE IF NOT EXISTS perf_events (
                event_time DateTime,
                event_type LowCardinality(String),
                user_id UInt32
            ) ENGINE = MergeTree()
            ORDER BY (event_type, event_time)
        </create_query>
        <fill_query>
            INSERT INTO perf_events
            SELECT now() - rand() % 86400, toString(rand() % 5), rand() % 100000
            FROM numbers(5000000)
        </fill_query>
    </preconditions>

    <query>SELECT count() FROM perf_events WHERE event_type = 'page_view'</query>
    <query>SELECT uniq(user_id) FROM perf_events WHERE event_time >= today()</query>
    <query>SELECT event_type, count() FROM perf_events GROUP BY event_type</query>

    <stop_conditions>
        <all_of>
            <iterations>50</iterations>
            <min_time_not_changing_for_ms>10000</min_time_not_changing_for_ms>
        </all_of>
    </stop_conditions>
</test>
```

## Comparing Two ClickHouse Versions

The framework supports side-by-side comparison:

```bash
python3 tests/performance/perf.py \
  --host localhost:9000 \
  --host new-server:9000 \
  --test tests/performance/my_test.xml \
  --output json > results.json
```

Then analyze:

```bash
python3 tests/performance/report.py results.json
```

## Interpreting Results

The framework reports:
- Median, 5th, and 95th percentile query times
- Speedup ratio between two servers
- Whether performance degraded beyond a threshold

```text
Query 1: 12.3 ms -> 9.8 ms (1.25x speedup)
Query 2: 45.1 ms -> 47.3 ms (0.95x, regression!)
```

## Using Query Log for Regression Analysis

After running the framework, query the performance log:

```sql
SELECT
    query,
    median(query_duration_ms) AS median_ms,
    count() AS runs
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time > now() - INTERVAL 30 MINUTE
GROUP BY query
ORDER BY median_ms DESC;
```

## Integrating into CI/CD

Run performance tests in your CI pipeline to catch regressions before deploying new ClickHouse versions:

```bash
#!/bin/bash
python3 perf.py --host staging:9000 --test my_test.xml --output json > staging.json
python3 perf.py --host production:9000 --test my_test.xml --output json > prod.json
python3 report.py --threshold 1.1 staging.json prod.json
```

## Summary

The ClickHouse performance tests framework provides a repeatable, structured way to benchmark queries and detect regressions. Write XML test files for your critical queries, run comparisons across ClickHouse versions or server configurations, and integrate into CI/CD to prevent performance degradation from reaching production.
