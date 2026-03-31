# How to Use MySQL Enterprise Monitor

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Enterprise, Monitoring, Performance

Description: Learn how MySQL Enterprise Monitor works, what it tracks, and how to use its key features to detect issues, optimize performance, and ensure database health.

---

## What Is MySQL Enterprise Monitor?

MySQL Enterprise Monitor (MEM) is a commercial monitoring solution included with MySQL Enterprise Edition. It provides real-time visibility into MySQL instance health, query performance, replication status, and security compliance through a web-based dashboard and automated advisors.

Unlike open-source alternatives, MEM integrates deeply with MySQL internals and provides curated best-practice rules (called advisors) that alert you when your configuration or usage deviates from recommended patterns.

## Architecture Overview

MySQL Enterprise Monitor consists of three components:

```text
[MySQL Instances] --> [MySQL Agent] --> [Service Manager]
                                              |
                                        [Web Dashboard]
```

- **MySQL Agent** - runs alongside each MySQL instance, collects metrics, and sends them to the Service Manager
- **Service Manager** - a central server that stores monitoring data in a MySQL repository database
- **Web Dashboard** - browser-based UI for viewing metrics, alerts, and query analysis

## Installing the Agent

```bash
# Download the agent installer from My Oracle Support
chmod +x mysqlmonitoragent-8.x.x-linux-installer.bin

# Run the installer
./mysqlmonitoragent-8.x.x-linux-installer.bin \
  --installdir /opt/mysql/enterprise/agent \
  --mysqlhost 127.0.0.1 \
  --mysqlport 3306 \
  --mysqluser mem_agent \
  --mysqlpassword 'AgentPass!2026' \
  --managerhost mem-server.internal \
  --managerport 18443
```

Create a dedicated agent user in MySQL:

```sql
CREATE USER 'mem_agent'@'127.0.0.1'
  IDENTIFIED BY 'AgentPass!2026'
  REQUIRE SSL;

GRANT REPLICATION CLIENT, PROCESS, SELECT
  ON *.* TO 'mem_agent'@'127.0.0.1';

GRANT SELECT ON performance_schema.*
  TO 'mem_agent'@'127.0.0.1';
```

## Key Monitoring Features

### Real-Time Metrics Dashboard

MEM tracks hundreds of server status variables and visualizes them over time:

```text
- Connections (active, waiting, refused)
- Query throughput (queries/sec, slow queries)
- InnoDB buffer pool hit ratio
- Replication lag (seconds behind source)
- Disk I/O and table cache usage
```

### Query Analyzer

The Query Analyzer captures slow and high-impact queries without enabling the general query log:

```sql
-- Verify the performance_schema is enabled
SHOW VARIABLES LIKE 'performance_schema';

-- Top queries by total execution time are surfaced automatically
-- No additional configuration needed with MEM agent active
```

### Advisors and Alerts

Advisors are automated rules that evaluate your instance configuration and flag potential issues:

```text
Example advisors:
- "Binary log not enabled" - warns if binlog is off
- "InnoDB buffer pool too small" - suggests buffer pool sizing
- "Users with no password" - security compliance check
- "Replication not running" - detects stopped replica threads
```

Configure alert thresholds from the MEM web UI or via the REST API:

```bash
# Example: query MEM API for active alerts
curl -k -u admin:password \
  https://mem-server:18443/rest/v1/advisor/alarms \
  -H "Accept: application/json"
```

## Monitoring Replication

```sql
-- MEM surfaces this data from SHOW REPLICA STATUS automatically
SHOW REPLICA STATUS\G
-- Key fields: Seconds_Behind_Source, Replica_IO_Running, Replica_SQL_Running
```

The dashboard graphs replication lag over time and alerts when it exceeds a configurable threshold.

## Summary

MySQL Enterprise Monitor provides deep, automated observability for MySQL deployments at scale. Its agents are lightweight, its advisors encode proven best practices, and the Query Analyzer helps identify and fix performance regressions without reading raw slow query logs. For teams running MySQL Enterprise Edition, MEM is the fastest path to proactive database health management.
