# How to Configure TimescaleDB with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: IPv6, TimescaleDB, PostgreSQL, Time Series Database, Monitoring

Description: Learn how to configure TimescaleDB (PostgreSQL extension) to accept connections over IPv6 addresses by configuring listen_addresses and pg_hba.conf.

## TimescaleDB IPv6 Overview

TimescaleDB is a PostgreSQL extension, so IPv6 configuration follows standard PostgreSQL procedures. The key files are `postgresql.conf` and `pg_hba.conf`.

## postgresql.conf Configuration

```ini
# /etc/postgresql/15/main/postgresql.conf

# Listen on specific IPv6 address

listen_addresses = '2001:db8::10'

# Listen on all interfaces (IPv4 and IPv6)
listen_addresses = '*'

# Listen on IPv6 loopback and localhost
listen_addresses = '::1,127.0.0.1'

# Port (default 5432)
port = 5432
```

## pg_hba.conf for IPv6 Clients

```text
# /etc/postgresql/15/main/pg_hba.conf

# Allow local IPv6 loopback connections
host    all             all             ::1/128                 scram-sha-256

# Allow connections from an IPv6 subnet
host    all             all             2001:db8::/32           scram-sha-256

# Allow specific IPv6 host
host    timescaledb     tsuser          2001:db8::20/128        scram-sha-256

# Allow all IPv6 connections (development only)
host    all             all             ::/0                    scram-sha-256
```

## Apply and Test

```bash
# Reload PostgreSQL configuration
systemctl reload postgresql

# Or use pg_ctl
pg_ctl reload -D /var/lib/postgresql/15/main

# Verify listening on IPv6
ss -6 -tlnp | grep postgres
# Expected: [2001:db8::10]:5432

# Test connection via psql over IPv6
psql -h 2001:db8::10 -p 5432 -U postgres -d postgres

# Check TimescaleDB extension is installed
psql -h 2001:db8::10 -U postgres -c "\dx timescaledb"
```

## Create TimescaleDB Hypertable over IPv6

```sql
-- Connect to database (psql over IPv6)
-- psql -h 2001:db8::10 -U postgres -d mydb

-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create a regular table
CREATE TABLE metrics (
    time        TIMESTAMPTZ NOT NULL,
    host        TEXT,
    metric_name TEXT,
    value       DOUBLE PRECISION
);

-- Convert to hypertable (TimescaleDB-specific)
SELECT create_hypertable('metrics', 'time');

-- Insert data
INSERT INTO metrics (time, host, metric_name, value)
VALUES (NOW(), 'server01', 'cpu_usage', 45.2);

-- Query time-series data
SELECT time_bucket('5 minutes', time) AS five_min,
       avg(value) AS avg_value
FROM metrics
WHERE metric_name = 'cpu_usage'
  AND time > NOW() - INTERVAL '1 hour'
GROUP BY five_min
ORDER BY five_min;
```

## Python Client over IPv6

```python
import psycopg2

# Connect to TimescaleDB via IPv6
conn = psycopg2.connect(
    host="2001:db8::10",
    port=5432,
    database="mydb",
    user="postgres",
    password="mypassword"
)

cursor = conn.cursor()

# Query hypertable
cursor.execute("""
    SELECT time_bucket('5 minutes', time) AS bucket,
           avg(value) AS avg_value
    FROM metrics
    WHERE metric_name = 'cpu_usage'
      AND time > NOW() - INTERVAL '1 hour'
    GROUP BY bucket
    ORDER BY bucket
""")

rows = cursor.fetchall()
for row in rows:
    print(f"Bucket: {row[0]}, Avg: {row[1]:.2f}")

cursor.close()
conn.close()
```

## Connection String Examples

```bash
# psql connection string
psql "host=2001:db8::10 port=5432 dbname=mydb user=postgres"

# JDBC URL (for Java/Grafana datasource)
jdbc:postgresql://[2001:db8::10]:5432/mydb

# SQLAlchemy URL (Python)
postgresql://postgres:password@[2001:db8::10]:5432/mydb
```

## Summary

TimescaleDB uses PostgreSQL's IPv6 configuration: set `listen_addresses = '2001:db8::10'` (or `'*'` for all interfaces) in `postgresql.conf`, and add IPv6 CIDR entries in `pg_hba.conf` (e.g., `host all all 2001:db8::/32 scram-sha-256`). Reload with `systemctl reload postgresql`. Connect with `psql -h 2001:db8::10`. TimescaleDB hypertables and continuous aggregates work transparently over IPv6 connections.
