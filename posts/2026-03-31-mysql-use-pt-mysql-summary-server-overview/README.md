# How to Use pt-mysql-summary for MySQL Server Overview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Diagnostic, Percona, Monitoring, Summary

Description: Learn how to use pt-mysql-summary to generate a comprehensive human-readable overview of a MySQL server's configuration, status, and health.

---

## What is pt-mysql-summary?

`pt-mysql-summary` is a Percona Toolkit utility that connects to a MySQL server and produces a detailed, structured report covering configuration, replication status, schema statistics, InnoDB metrics, and operating system information. It is one of the first tools to run when diagnosing an unfamiliar MySQL server or troubleshooting a production issue, because it consolidates dozens of queries and system commands into a single readable snapshot.

## Basic Usage

```bash
pt-mysql-summary -- --host=127.0.0.1 --user=root --password=secret
```

Note the `--` separator: arguments before it are passed to `pt-mysql-summary` itself, and arguments after it are passed to the `mysql` client it invokes internally.

## Output Sections

The report is divided into clearly labeled sections:

```text
# Percona Toolkit MySQL Summary Report #######################
# System #####################################################
# MySQL Executable ############################################
# Report On Port 3306 #########################################
# MySQL System Variables (notable ones) ######################
# MySQL Status Counters #######################################
# InnoDB #####################################################
# Binary Logging #############################################
# Noteworthy Variables ########################################
# Replication #################################################
# Table Schemas ###############################################
```

## Key Sections Explained

The `# MySQL System Variables` section highlights the most important tuning parameters:

```text
            version | 8.0.36
    innodb_buffer_pool_size | 4294967296
         max_connections | 500
        slow_query_log | ON
   innodb_file_per_table | ON
              log_bin | ON
```

The `# Replication` section shows replication health at a glance:

```text
Replication
               Slave running: No
              Master running: Yes
            Binary log files: 42
         Binary log position: 104857600
```

## Saving the Report

Always save the output for future reference and incident documentation:

```bash
pt-mysql-summary -- \
  --host=127.0.0.1 \
  --user=root \
  --password=secret \
  > mysql_summary_$(date +%Y%m%d_%H%M%S).txt
```

## Remote Server Analysis

```bash
pt-mysql-summary -- \
  --host=db.prod.internal \
  --port=3306 \
  --user=dba_admin \
  --password=secret
```

## Analyzing Replication Replicas

Run separately on each replica to capture their individual status:

```bash
# On the primary
pt-mysql-summary -- --host=primary.db --user=root --password=secret > primary_summary.txt

# On the replica
pt-mysql-summary -- --host=replica.db --user=root --password=secret > replica_summary.txt
```

Compare the reports to identify configuration differences that could affect replication performance.

## Schema Statistics Section

The tool also reports on database sizes and table counts:

```text
# Schema #####################################################
  Database  Tables   Views  SPs Trigs  Funcs    FKs  Partn
    mydb      142       8    12     5      3     89      0
    archive    18       0     0     0      0      5      0
```

This is useful for capacity planning and understanding schema complexity.

## Using with pt-variable-advisor

For a complete server review, combine both tools:

```bash
pt-mysql-summary -- --host=127.0.0.1 --user=root --password=secret
echo "---VARIABLE ADVISOR---"
pt-variable-advisor --host=127.0.0.1 --user=root --password=secret
```

## Summary

`pt-mysql-summary` is the first tool to run on any MySQL server you need to understand quickly. It generates a complete health snapshot in seconds, covering everything from buffer pool settings to replication lag to schema statistics. Keep saved reports from regular runs to track how the server changes over time and to provide baseline context during incident response.
