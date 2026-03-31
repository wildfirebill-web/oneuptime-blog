# How to Configure the InfluxDB Module in Ceph Manager

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, InfluxDB, Monitoring, Time Series

Description: Learn how to enable and configure the Ceph Manager InfluxDB module to push cluster performance metrics to an InfluxDB time-series database.

---

The Ceph Manager InfluxDB module continuously pushes cluster performance metrics to an InfluxDB instance. This is useful when your observability stack is already built around InfluxDB and Grafana, allowing you to view Ceph metrics alongside other infrastructure data.

## Enabling the Module

Enable the InfluxDB module using the `ceph mgr module` command:

```bash
ceph mgr module enable influx
```

Verify it is loaded:

```bash
ceph mgr module ls | grep influx
```

## Configuring the InfluxDB Connection

Set the InfluxDB server hostname and port:

```bash
ceph config set mgr mgr/influx/hostname influxdb.example.com
ceph config set mgr mgr/influx/port 8086
```

Specify the database name and credentials:

```bash
ceph config set mgr mgr/influx/database ceph
ceph config set mgr mgr/influx/username ceph_user
ceph config set mgr mgr/influx/password secret123
```

## Controlling the Push Interval

By default, metrics are pushed every 30 seconds. Adjust this to reduce overhead or increase resolution:

```bash
ceph config set mgr mgr/influx/interval 10
```

## Verifying Data in InfluxDB

Once configured, connect to InfluxDB and query incoming data:

```bash
influx -host influxdb.example.com -database ceph
```

```sql
SHOW MEASUREMENTS;
SELECT * FROM ceph_osd_stats LIMIT 10;
```

Example measurement fields you will see:

```
time                apply_latency_ms commit_latency_ms osd_id
2026-03-31T00:00:00Z 1.2             2.5               0
2026-03-31T00:00:00Z 0.8             1.9               1
```

## Grafana Dashboard

Import the official Ceph InfluxDB dashboard from Grafana's dashboard library or build custom panels using these measurements:

- `ceph_osd_stats` - Per-OSD apply and commit latency
- `ceph_pool_stats` - Pool read/write IOPS and bandwidth
- `ceph_cluster_stats` - Global health, capacity, and PG states

```bash
# Verify metric ingestion rate
influx -execute "SELECT count(*) FROM ceph_osd_stats WHERE time > now() - 5m" -database ceph
```

## Disabling the Module

To stop sending data to InfluxDB:

```bash
ceph mgr module disable influx
```

## Summary

The Ceph Manager InfluxDB module provides a push-based integration that sends cluster metrics directly to InfluxDB at a configurable interval. Configuration is straightforward using `ceph config set`, and once running, rich time-series data about OSDs, pools, and cluster health is available for Grafana dashboards and alerting.
