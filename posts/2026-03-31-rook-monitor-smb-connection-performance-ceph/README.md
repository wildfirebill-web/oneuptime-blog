# How to Monitor SMB Connection Performance with Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, SMB, Monitoring, Performance

Description: Learn how to monitor SMB connection performance for Ceph-backed Samba shares using smbstatus, Samba VFS statistics, and Prometheus metrics.

---

## Monitoring Layers

SMB performance monitoring for Ceph spans three layers:
- Samba session and connection statistics
- CephFS I/O metrics at the MDS and OSD level
- System resource utilization on gateway nodes

## Using smbstatus

`smbstatus` shows active connections, open files, and file locks:

```bash
# Show all active sessions
smbstatus

# Show only connections
smbstatus -S

# Show open files
smbstatus -L

# Show file locks
smbstatus -B
```

Example output:

```
Samba version 4.19.0
PID     Username     Group        Machine    Protocol Version  Encryption  Signing
------------------------------------------------------------------------
12345   alice        engineers    10.0.1.100 SMB3_11  -          -

Service      pid     Machine       Connected at                     Encryption  Signing
---------------------------------------------------------------------------
cephshare    12345   10.0.1.100    Tue Mar 31 09:15:12 2026 UTC     -          -
```

## Monitoring with net status

The `net` command provides summary statistics:

```bash
net status sessions
net status shares
```

## CephFS Performance Statistics

Check MDS performance counters relevant to SMB workloads:

```bash
# MDS operation counts
ceph daemon mds.* perf dump | python3 -m json.tool | grep -A2 "req_"

# Client I/O statistics
ceph fs status
```

Monitor active CephFS clients:

```bash
ceph tell mds.0 client ls | python3 -m json.tool | grep -E "client_id|hostname"
```

## Collecting Samba Metrics with Prometheus

Use `samba-exporter` to expose Samba metrics to Prometheus:

```bash
pip3 install samba-prometheus-exporter
samba-exporter --port 9922 &
```

Key metrics exposed:

```
samba_active_connections_total
samba_open_files_total
samba_bytes_read_total
samba_bytes_written_total
samba_calls_total{operation="SMBwrite"}
```

Scrape configuration for Prometheus:

```yaml
scrape_configs:
  - job_name: 'samba'
    static_configs:
      - targets: ['samba01:9922', 'samba02:9922']
```

## Grafana Dashboard Panels

Create panels for key SMB metrics:

```
# Active connections over time
samba_active_connections_total

# Write throughput in MB/s
rate(samba_bytes_written_total[5m]) / 1024 / 1024

# Read throughput in MB/s
rate(samba_bytes_read_total[5m]) / 1024 / 1024

# Operations per second
rate(samba_calls_total[1m])
```

## Identifying Slow Operations

Enable the slow query log in Samba:

```ini
[global]
    log level = 1 smb2:10
    smb2 leases = yes
```

Check for slow operations in the log:

```bash
grep "slow" /var/log/samba/log.smbd | tail -20
```

Monitor CephFS latency for underlying operations:

```bash
ceph daemon mds.0 perf dump | python3 -c "
import sys, json
d = json.load(sys.stdin)
lat = d.get('mds_server', {}).get('req_setattr_latency', {})
print(f'setattr avg latency: {lat.get(\"avgcount\", 0)}ms')
"
```

## Summary

Monitoring SMB performance for Ceph involves using `smbstatus` for real-time session visibility, CephFS metrics for storage-layer latency, and Prometheus with a Samba exporter for historical trending. Combining these sources in Grafana dashboards provides the visibility needed to identify whether performance issues originate at the SMB gateway, the CephFS metadata layer, or the underlying OSD storage.
