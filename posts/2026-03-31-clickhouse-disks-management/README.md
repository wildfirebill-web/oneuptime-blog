# How to Use clickhouse-disks for Disk Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Disk, CLI, Management, Storage, Administration

Description: Learn how to use the clickhouse-disks command-line tool to inspect, copy, move, and remove files across all configured ClickHouse storage disks without stopping the server.

---

## Introduction

`clickhouse-disks` is a standalone CLI utility included with ClickHouse that lets administrators interact with all configured disks (local, S3, GCS, Azure Blob) at the file level. It is useful for manual data migrations, disaster recovery, space audits, and debugging storage configuration without writing SQL.

## Installation

`clickhouse-disks` ships as part of the `clickhouse-server` package:

```bash
which clickhouse-disks
# /usr/bin/clickhouse-disks
```

## Getting Help

```bash
clickhouse-disks --help
```

## Listing Configured Disks

```bash
clickhouse-disks list-disks \
    --config-file /etc/clickhouse-server/config.xml
```

```
default
s3_cold
s3_cache
azure_cold
```

## Listing Files on a Disk

```bash
# List the top-level directories on the default disk
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    --disk default \
    list /

# List a specific table's data directory
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    --disk default \
    list /store/abc/def/events/
```

## Checking Disk Usage

```bash
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    --disk default \
    disk-usage /
```

## Copying Files Between Disks

Copy a table's data directory from local disk to S3:

```bash
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    copy \
    --disk-from default \
    --disk-to s3_cold \
    /store/abc/def/events/ \
    /store/abc/def/events/
```

This is useful for manual cold migration outside of TTL or storage policy rules.

## Moving Files

```bash
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    move \
    --disk-from default \
    --disk-to s3_cold \
    /store/abc/def/events/20240101_1_1_0/ \
    /store/abc/def/events/20240101_1_1_0/
```

## Removing Files

```bash
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    --disk s3_cold \
    remove /store/abc/def/events/20230101_1_1_0/
```

## Reading a File (Debugging)

Inspect the content of a metadata or checksum file:

```bash
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    --disk default \
    read /store/abc/def/events/20240101_1_1_0/checksums.txt
```

## Writing a File

```bash
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    --disk default \
    write /store/test/marker.txt <<< "test"
```

## Interactive Shell Mode

`clickhouse-disks` supports an interactive REPL for browsing multiple disks in one session:

```bash
clickhouse-disks \
    --config-file /etc/clickhouse-server/config.xml \
    --disk default \
    --interactive
```

Inside the shell:

```
> ls /
> cd store/abc/def/events/
> ls
> copy 20240101_1_1_0/ --disk-to s3_cold --path-to 20240101_1_1_0/
> quit
```

## Common Use Cases

| Use case | Command |
|---|---|
| Audit how much space a table uses on S3 | `disk-usage /store/...` on `s3_cold` |
| Manually migrate cold partitions | `copy --disk-from default --disk-to s3_cold` |
| Verify a file exists on S3 | `list /store/.../part-name/` on `s3_cold` |
| Clean up orphaned files | `remove /store/.../old-part/` |

## Summary

`clickhouse-disks` is a file-level CLI tool for managing all ClickHouse disks without connecting to the server via SQL. Use it to list, copy, move, read, write, and delete files across local, S3, GCS, and Azure Blob disks. It is particularly useful for manual partition migrations, storage audits, and disaster recovery scenarios where SQL-level access is unavailable.
