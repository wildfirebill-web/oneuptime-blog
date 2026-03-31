# What Is Percona Monitoring and Management (PMM) for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Percona PMM, Monitoring, Observability, Performance Schema, Grafana

Description: Percona Monitoring and Management (PMM) is a free open-source platform that provides deep observability into MySQL performance using Grafana dashboards and Query Analytics.

---

## Overview

Percona Monitoring and Management (PMM) is an open-source database observability platform built by Percona. It combines Grafana dashboards, Prometheus metrics, and a unique Query Analytics (QAN) engine to give database administrators deep visibility into MySQL performance, query patterns, and server health - all in a single unified interface.

## Architecture

```text
                  PMM Server (Docker)
                  +------------------+
                  | Grafana (3000)   |
                  | Prometheus       |
                  | Query Analytics  |
                  | ClickHouse       |
                  +------------------+
                         ^
                         |
              PMM Client (on each DB host)
              +---------------------------+
              | pmm-agent                 |
              | - mysqld_exporter         |
              | - query analytics agent   |
              +---------------------------+
                         |
                    MySQL Server
```

## Installing PMM Server

The simplest deployment uses Docker:

```bash
# Pull and start PMM Server
docker pull percona/pmm-server:2
docker run -d \
  --name pmm-server \
  -p 443:443 \
  -p 80:80 \
  -v pmm-data:/srv \
  percona/pmm-server:2

# Access at https://your-server (default: admin/admin)
```

## Installing PMM Client

```bash
# On RHEL/CentOS
sudo yum install pmm2-client

# On Debian/Ubuntu
sudo apt install pmm2-client

# Configure client to connect to PMM Server
pmm-admin config --server-insecure-tls --server-url=https://admin:admin@pmm-server-host
```

## Adding a MySQL Instance

```sql
-- Create a dedicated PMM user in MySQL
CREATE USER 'pmm'@'localhost' IDENTIFIED BY 'pmmpass' WITH MAX_USER_CONNECTIONS 10;
GRANT SELECT, PROCESS, SUPER, REPLICATION CLIENT, RELOAD ON *.* TO 'pmm'@'localhost';
GRANT SELECT, UPDATE, DELETE, DROP ON performance_schema.* TO 'pmm'@'localhost';
FLUSH PRIVILEGES;
```

```bash
# Register the MySQL instance with PMM
pmm-admin add mysql \
  --username=pmm \
  --password=pmmpass \
  --service-name=mysql-primary \
  --host=localhost \
  --port=3306 \
  --query-source=perfschema
```

## Key Dashboards

PMM ships with pre-built Grafana dashboards covering:

### MySQL Overview

Tracks the core health metrics:

```text
- Queries per second (QPS) by type (SELECT/INSERT/UPDATE/DELETE)
- InnoDB buffer pool hit ratio
- Threads connected vs. running
- Replication lag
- Disk I/O utilization
```

### Query Analytics (QAN)

QAN captures per-query performance data, aggregated by digest:

```text
Query: SELECT * FROM orders WHERE customer_id = ?
Calls:    12,450 (last 1 hour)
Avg time: 45ms
P99 time: 230ms
Rows examined: 850 avg
No index used: YES  <-- needs attention
```

This helps identify:
- Queries missing indexes (high rows-examined-to-rows-sent ratio)
- Queries with high P99 latency
- Most frequently executed queries

### InnoDB Internals Dashboard

```text
- Buffer pool data pages
- Row operations (inserted/updated/deleted/read)
- Lock waits per second
- Dirty pages ratio
- Checkpoint age
```

## Setting Up Alerts

PMM integrates with Grafana's alerting system:

```text
Alert: High Replication Lag
Condition: mysql_slave_lag_seconds > 30
For: 5 minutes
Severity: Critical
Notification: Slack #dba-alerts
```

```bash
# View current PMM-managed services
pmm-admin list

# Sample output:
# Service type  Service name    Address and port  Service ID
# MySQL         mysql-primary   127.0.0.1:3306    /service_id/abc123
# MySQL         mysql-replica1  10.0.1.11:3306    /service_id/def456
```

## Performance Schema Configuration

PMM works best with Performance Schema fully enabled:

```sql
-- Verify Performance Schema is on
SHOW VARIABLES LIKE 'performance_schema';

-- Enable key consumers
UPDATE performance_schema.setup_consumers
SET enabled = 'YES'
WHERE name IN (
    'events_statements_history_long',
    'events_waits_history_long'
);

-- Enable key instruments
UPDATE performance_schema.setup_instruments
SET enabled = 'YES', timed = 'YES'
WHERE name LIKE 'statement/%';
```

## Removing a Service

```bash
# Stop monitoring a MySQL instance
pmm-admin remove mysql mysql-primary

# List remaining services
pmm-admin list
```

## PMM vs Other Tools

| Feature | PMM | MySQLTuner | Prometheus + mysqld_exporter |
|---------|-----|------------|------------------------------|
| Query Analytics | Yes | No | No |
| Pre-built dashboards | 30+ | No | Manual |
| Historical data | Yes | No | Yes |
| Advisor/recommendations | Yes | Yes | No |
| Setup complexity | Medium | Low | Medium |

## Summary

Percona Monitoring and Management is a comprehensive observability solution for MySQL that goes well beyond basic metric collection. Its Query Analytics engine surfaces slow and problematic queries from real traffic, while its Grafana dashboards provide instant visibility into InnoDB internals, replication status, and resource utilization. PMM is the recommended monitoring solution for any MySQL environment where query-level performance visibility and historical trending are required.
