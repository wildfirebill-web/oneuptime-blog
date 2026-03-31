# How to Set Up Audit Trails for Ceph Storage Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Audit, Compliance, Security, Logging

Description: Learn how to enable and configure audit trails for Ceph storage access to track who accessed what data and when, supporting compliance requirements.

---

Audit trails are essential for any storage system handling sensitive data. Ceph provides several mechanisms to log access events across RGW (object storage), CephFS, and RBD, giving you a complete picture of data access.

## Enabling RGW Request Logging

The Ceph RADOS Gateway supports detailed HTTP request logging. Enable it in your Rook CephObjectStore:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  gateway:
    port: 80
    instances: 1
  zone:
    name: my-zone
```

Then configure RGW logging via the Ceph config:

```bash
ceph config set client.rgw rgw_enable_ops_log true
ceph config set client.rgw rgw_ops_log_socket_path /var/log/ceph/rgw-ops.log
ceph config set client.rgw rgw_log_http_headers "CONTENT_TYPE,HTTP_X_FORWARDED_FOR"
```

## Configuring Usage Logging for Object Access

Enable per-user usage logging to track S3 API calls:

```bash
ceph config set global rgw_enable_usage_log true
ceph config set global rgw_usage_log_tick_interval 30
ceph config set global rgw_usage_log_flush_threshold 1024
```

View usage logs with:

```bash
radosgw-admin usage show --uid=myuser --start-date=2026-01-01 --end-date=2026-03-31
```

## Forwarding Logs to a SIEM

For centralized audit trail management, forward Ceph logs to a SIEM like Elasticsearch. Configure Fluentd to collect RGW logs:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-ceph-config
  namespace: rook-ceph
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/ceph/ceph-rgw*.log
      pos_file /var/log/fluentd/ceph-rgw.pos
      tag ceph.rgw
      <parse>
        @type json
      </parse>
    </source>
    <match ceph.**>
      @type elasticsearch
      host elasticsearch.logging.svc
      port 9200
      index_name ceph-audit
    </match>
```

## CephFS Audit Logging

For CephFS, enable MDS audit logging to capture filesystem operations:

```bash
ceph config set mds log_to_file true
ceph config set mds mds_op_history_size 20
ceph config set mds mds_log_max_segments 128
```

Query MDS audit events via the admin socket:

```bash
ceph daemon mds.0 dump_historic_ops
```

## Parsing and Alerting on Audit Logs

Create a simple script to alert on suspicious access patterns:

```python
import json
import subprocess

def check_failed_auth():
    result = subprocess.run(
        ["radosgw-admin", "log", "list"],
        capture_output=True, text=True
    )
    logs = json.loads(result.stdout)
    for entry in logs:
        if entry.get("http_status") == "403":
            print(f"ALERT: Unauthorized access by {entry['user']} at {entry['time']}")

check_failed_auth()
```

## Summary

Ceph provides robust audit trail capabilities through RGW ops logging, usage logs, and MDS audit events. By forwarding these logs to a centralized SIEM and implementing alerting scripts, you can maintain a complete record of storage access for compliance and security investigations.
