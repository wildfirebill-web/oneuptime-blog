# ClickHouse vs DuckDB for Embedded Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DuckDB, Comparison, Embedded Analytics, OLAP

Description: Compare ClickHouse and DuckDB for analytical workloads, covering embedded vs server deployment, performance, and the scenarios where each excels.

---

## Overview

ClickHouse and DuckDB are both columnar OLAP engines, but they serve fundamentally different deployment models. ClickHouse is a server-based database for shared analytical infrastructure, while DuckDB is an embedded in-process database like SQLite but for analytics.

## Deployment Model

### ClickHouse - Server Process

```bash
# ClickHouse runs as a dedicated server
docker run -d --name clickhouse \
    -p 8123:8123 -p 9000:9000 \
    clickhouse/clickhouse-server
```

```python
from clickhouse_driver import Client
client = Client(host='localhost')
result = client.execute('SELECT count() FROM events')
```

### DuckDB - Embedded In-Process

```python
import duckdb

# DuckDB runs inside the application process
conn = duckdb.connect(database='analytics.duckdb')
result = conn.execute('SELECT count(*) FROM events').fetchall()
conn.close()
```

No separate server process, no network overhead, no configuration.

## Query Performance

Both use columnar vectorized execution. DuckDB uses an out-of-core execution model that can handle datasets larger than RAM efficiently.

### ClickHouse Example Query

```sql
SELECT
    toStartOfDay(ts) AS day,
    count() AS events,
    uniq(user_id) AS users
FROM events
WHERE ts >= today() - 30
GROUP BY day
ORDER BY day;
```

### DuckDB Equivalent

```python
conn.execute("""
    SELECT
        date_trunc('day', ts) AS day,
        count(*) AS events,
        count(DISTINCT user_id) AS users
    FROM events
    WHERE ts >= current_date - INTERVAL 30 DAY
    GROUP BY day
    ORDER BY day
""").fetchdf()
```

## Querying Files Directly

DuckDB excels at querying files directly without loading them first:

```python
import duckdb

# Query Parquet files directly
conn = duckdb.connect()
result = conn.execute("""
    SELECT
        date_trunc('month', ts) AS month,
        sum(amount) AS revenue
    FROM 'events_*.parquet'
    GROUP BY month
    ORDER BY month
""").fetchdf()
```

ClickHouse can also query files using table functions:

```sql
SELECT
    toStartOfMonth(ts) AS month,
    sum(amount) AS revenue
FROM s3('s3://bucket/events_*.parquet', 'Parquet')
GROUP BY month
ORDER BY month;
```

## Concurrency and Multi-User Access

ClickHouse handles many concurrent queries from many users:

```text
- Multiple client connections
- Query queue management
- User-level quotas and resource limits
- Distributed query execution
```

DuckDB is single-writer / limited-concurrency:

```python
# DuckDB with concurrent reads - works with multiple connections in read mode
conn1 = duckdb.connect('analytics.duckdb', read_only=True)
conn2 = duckdb.connect('analytics.duckdb', read_only=True)
# Only ONE write connection at a time is allowed
```

## Scale Comparison

| Dimension | ClickHouse | DuckDB |
|---|---|---|
| Dataset size | Petabytes (cluster) | Hundreds of GB (single node) |
| Concurrency | Thousands of queries | Low concurrency |
| Deployment | Server (requires ops) | In-process (zero ops) |
| Network overhead | Yes (client/server) | None |
| Use case | Shared analytics | Single-app analytics |
| Data freshness | Streaming + batch | File-based or batch |

## Python Data Science Workflow with DuckDB

```python
import duckdb
import pandas as pd

conn = duckdb.connect()

# Query a Pandas DataFrame directly
df = pd.DataFrame({'user_id': range(1000000), 'value': range(1000000)})

result = conn.execute("""
    SELECT
        user_id % 100 AS bucket,
        avg(value) AS avg_val
    FROM df
    GROUP BY bucket
    ORDER BY bucket
""").fetchdf()
```

## When to Choose ClickHouse

- Shared cluster for multiple teams or applications
- Real-time streaming ingestion from Kafka
- Large data volumes (terabytes to petabytes)
- Need for multi-user access control and quotas
- Centralized analytics infrastructure

## When to Choose DuckDB

- Embedded analytics within a single application
- Data science notebooks and exploration
- Querying local Parquet/CSV/JSON files
- Single-developer or small-team workflows
- No desire to manage a separate server process
- Processing files on a local machine or Lambda function

## Summary

ClickHouse and DuckDB solve different problems. ClickHouse is the right choice for a shared analytics database with many concurrent users, streaming ingestion, and petabyte-scale data. DuckDB is ideal for in-process analytics embedded directly in an application, data science workflows over local files, and scenarios where zero infrastructure overhead is a priority. They are complementary - many teams use DuckDB for local exploration and ClickHouse for production serving.
