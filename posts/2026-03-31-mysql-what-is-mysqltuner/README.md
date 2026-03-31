# What Is MySQLTuner

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MySQLTuner, Performance Tuning, Configuration, DBA Tools

Description: MySQLTuner is a Perl script that analyzes a running MySQL server's configuration and status variables, then provides prioritized recommendations to improve performance.

---

## Overview

MySQLTuner is a widely-used open-source Perl script that audits a running MySQL server and generates a report of potential performance improvements. It checks hundreds of configuration variables against runtime statistics to identify misconfigurations, undersized buffers, and tuning opportunities - all without modifying anything.

## Installation

```bash
# Download directly
wget https://raw.githubusercontent.com/major/MySQLTuner-perl/master/mysqltuner.pl
chmod +x mysqltuner.pl

# Or install via package manager (some distros)
sudo yum install mysqltuner    # RHEL/CentOS
sudo apt install mysqltuner    # Debian/Ubuntu
```

## Basic Usage

```bash
# Run with root credentials (interactive password prompt)
perl mysqltuner.pl --user root --host 127.0.0.1

# Or supply password directly (less secure, for automation)
perl mysqltuner.pl --user root --pass yourpassword --host 127.0.0.1

# Save report to a file
perl mysqltuner.pl --user root --outputfile /tmp/mysql_tuning_report.txt
```

## Sample Output

```text
 >>  MySQLTuner 2.5.2 - Major Hayden <major@mhtx.net>
 >>  Bug reports, feature requests, and downloads at http://mysqltuner.com/
 >>  Run with '--help' for additional options and output filtering

[--] Performing tests on 127.0.0.1:3306...
[OK] Logged in as root@127.0.0.1
[OK] Currently running supported MySQL version 8.0.35-MySQL Community Server

-------- Log File Analysis -------------------------------------------------
[OK] Log error file : /var/log/mysql/error.log
[OK] Log error file is not empty

-------- Storage Engine Statistics -----------------------------------------
[--] Status: +ARCHIVE +BLACKHOLE +CSV -FEDERATED +InnoDB +MEMORY +MRG_MYISAM +MyISAM
[--] Data in InnoDB tables: 2.5G (Tables: 142)
[OK] Total fragmented tables: 0

-------- Security Recommendations ------------------------------------------
[OK] All database users have passwords assigned
[!!] User 'devuser'@'%' has no restriction for host

-------- Performance Metrics -----------------------------------------------
[--] Up for: 7d 4h 23m 15s (45,678,234 q [75.3 qps], 1,234 conn)
[OK] Slow queries: 0% (0/45,678,234)
[!!] Highest connection usage: 87% (87/100)
[OK] Key buffer size / total MyISAM indexes: 8.0M/0B
[OK] InnoDB buffer pool / data size: 1.0G/2.5G
[!!] InnoDB buffer pool size is less than data size

-------- Recommendations ---------------------------------------------------
General recommendations:
    Increase innodb_buffer_pool_size to at least 75% of data size (3.2G)
    Consider raising max_connections from 100 to 200

Variables to adjust:
    innodb_buffer_pool_size (>= 3.2G)
    max_connections (> 100)
    innodb_log_file_size (current: 48M, recommended: 256M)
```

## Key Sections Explained

### InnoDB Buffer Pool

The most impactful recommendation is usually the InnoDB buffer pool size. A larger pool means more data cached in RAM, reducing disk I/O:

```sql
-- Check current buffer pool utilization
SELECT
    ROUND(
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') /
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total') * 100, 2
    ) AS buffer_pool_fill_pct;
```

### Connection Usage

If connection usage is high, you risk "Too many connections" errors:

```sql
SHOW STATUS LIKE 'Max_used_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';
```

### Slow Query Rate

MySQLTuner flags if slow queries exceed 5% of total:

```sql
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
SHOW STATUS LIKE 'Slow_queries';
```

## Acting on MySQLTuner Recommendations

After reviewing the output, apply changes to `/etc/mysql/my.cnf` or `/etc/my.cnf`:

```text
[mysqld]
# Increase buffer pool (set to ~75% of RAM for dedicated MySQL servers)
innodb_buffer_pool_size = 4G
innodb_buffer_pool_instances = 4

# Increase redo log capacity
innodb_redo_log_capacity = 536870912

# Raise connection limit
max_connections = 300

# Enable slow query logging
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

After editing, restart MySQL:

```bash
systemctl restart mysqld

# Verify settings took effect
mysql -u root -p -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
```

## Running on a Schedule

For ongoing monitoring, run MySQLTuner weekly and track recommendations over time:

```bash
# Add to crontab
0 2 * * 0 perl /usr/local/bin/mysqltuner.pl \
  --user root \
  --pass "$(cat /etc/mysql/.root_pass)" \
  --outputfile /var/log/mysqltuner-$(date +\%Y\%m\%d).log 2>&1
```

## Limitations

MySQLTuner gives general guidance but is not a replacement for workload-specific tuning. Its recommendations are based on heuristics and global statistics - for accurate advice:

- Let the server run for at least 24 hours of representative traffic before running MySQLTuner
- Use `EXPLAIN` and the Performance Schema for query-level tuning
- Consider workload type (OLTP vs analytics) when evaluating buffer size recommendations

## Summary

MySQLTuner is a quick, low-risk diagnostic tool that provides a prioritized list of configuration improvements for any MySQL server. It is most useful as a first-pass health check on new installations or when troubleshooting performance degradation. Its recommendations around buffer pool sizing, connection limits, and slow query logging cover the most common performance issues and are a reliable starting point for MySQL tuning efforts.
