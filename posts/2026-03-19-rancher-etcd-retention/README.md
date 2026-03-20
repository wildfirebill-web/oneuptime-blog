# How to Configure etcd Snapshot Retention in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, etcd, Backup

Description: Learn how to configure etcd snapshot retention policies in Rancher to balance storage usage with recovery point coverage.

etcd snapshots can accumulate quickly and consume significant disk space if retention is not properly configured. Rancher lets you control how many snapshots to keep and how often they are taken for both RKE and RKE2 clusters. This guide covers configuring retention policies to balance storage with recovery needs.

## Prerequisites

- Rancher v2.5 or later
- RKE or RKE2 managed clusters
- Admin access to Rancher

## Step 1: Understand Retention Settings

Rancher provides two key settings for snapshot management:

- **Snapshot Interval/Schedule**: How often snapshots are taken (e.g., every 6 hours).
- **Snapshot Retention**: How many snapshots are kept before older ones are automatically deleted.

The combination determines your recovery window. For example, snapshots every 6 hours with a retention of 5 gives you a 30-hour recovery window.

## Step 2: Configure Retention for RKE2 Clusters via UI

1. In Rancher, go to **Cluster Management**.
2. Find the RKE2 cluster and click the three-dot menu.
3. Select **Edit Config**.
4. Navigate to the **etcd** section.
5. Set **Snapshot Schedule Cron** to your desired interval:
   - `0 */6 * * *` for every 6 hours
   - `0 */12 * * *` for every 12 hours
   - `0 0 * * *` for daily
6. Set **Snapshot Retention** to the number of snapshots to keep.
7. Click **Save**.

## Step 3: Configure Retention for RKE2 Clusters via YAML

Edit the cluster resource directly:

```yaml
apiVersion: provisioning.cattle.io/v1
kind: Cluster
metadata:
  name: my-cluster
  namespace: fleet-default
spec:
  rkeConfig:
    etcd:
      snapshotScheduleCron: "0 */6 * * *"
      snapshotRetention: 10
      disableSnapshots: false
```

Apply the changes:

```bash
kubectl apply -f cluster.yaml
```

## Step 4: Configure Retention for RKE Clusters

For RKE clusters, update the cluster configuration through the Rancher API or UI:

```yaml
services:
  etcd:
    snapshot: true
    creation: "6h"
    retention: "72h"
    backup_config:
      enabled: true
      interval_hours: 6
      retention: 12
```

The `retention` field in `backup_config` specifies the number of snapshots to keep. The `retention` under `etcd` specifies the time duration.

## Step 5: Configure S3 Retention Separately

When using S3 for etcd snapshots, you can set separate retention for local and S3 snapshots:

```yaml
spec:
  rkeConfig:
    etcd:
      snapshotScheduleCron: "0 */6 * * *"
      snapshotRetention: 5
      s3:
        bucket: etcd-snapshots
        region: us-east-1
        endpoint: s3.amazonaws.com
        cloudCredentialName: s3-credential
```

The `snapshotRetention` applies to both local and S3 snapshots. For additional S3-level retention, use S3 lifecycle policies:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket etcd-snapshots \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "RetainSnapshots",
        "Status": "Enabled",
        "Filter": {"Prefix": ""},
        "Expiration": {"Days": 90},
        "Transitions": [
          {"Days": 30, "StorageClass": "STANDARD_IA"},
          {"Days": 60, "StorageClass": "GLACIER"}
        ]
      }
    ]
  }'
```

## Step 6: Retention Sizing Guidelines

Use this table to determine appropriate retention settings based on your requirements:

| RPO Target | Snapshot Interval | Retention Count | Recovery Window |
|------------|------------------|-----------------|-----------------|
| 1 hour | `0 * * * *` | 24 | 24 hours |
| 4 hours | `0 */4 * * *` | 12 | 48 hours |
| 6 hours | `0 */6 * * *` | 10 | 60 hours |
| 12 hours | `0 */12 * * *` | 14 | 7 days |
| 24 hours | `0 0 * * *` | 30 | 30 days |

Consider these factors when choosing settings:

- **Disk space**: Each snapshot can range from a few MB to several GB depending on cluster size.
- **Recovery window**: Longer windows provide more rollback options but use more storage.
- **Compliance**: Some regulations require minimum retention periods.

## Step 7: Monitor Disk Usage

Check how much space etcd snapshots are consuming on control plane nodes:

```bash
# RKE2

du -sh /var/lib/rancher/rke2/server/db/snapshots/

# RKE
du -sh /opt/rke/etcd-snapshots/
```

Set up an alert for disk space:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-disk-alerts
  namespace: cattle-monitoring-system
spec:
  groups:
  - name: etcd-storage
    rules:
    - alert: EtcdSnapshotDiskHigh
      expr: |
        (node_filesystem_avail_bytes{mountpoint="/var/lib/rancher"} /
         node_filesystem_size_bytes{mountpoint="/var/lib/rancher"}) < 0.15
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "etcd snapshot disk usage above 85%"
```

## Step 8: Clean Up Old Snapshots Manually

If you need to manually clean up snapshots that were not deleted by the retention policy:

### On RKE2 Nodes

```bash
# List snapshots sorted by date
ls -lt /var/lib/rancher/rke2/server/db/snapshots/

# Delete specific old snapshots
rm /var/lib/rancher/rke2/server/db/snapshots/etcd-snapshot-old-date*
```

### Via the Rancher UI

1. Navigate to the cluster.
2. Go to **Snapshots**.
3. Select the snapshots you want to delete.
4. Click **Delete**.

### In S3

```bash
aws s3 rm s3://etcd-snapshots/cluster-1/ --recursive \
  --exclude "*" --include "etcd-snapshot-2026-01*"
```

## Step 9: Verify Retention Is Working

After the retention policy has had time to take effect, verify that old snapshots are being cleaned up:

```bash
# Check local snapshot count
ls /var/lib/rancher/rke2/server/db/snapshots/ | wc -l

# Check S3 snapshot count
aws s3 ls s3://etcd-snapshots/cluster-1/ | wc -l
```

The counts should not exceed your configured retention value (plus a small buffer for in-progress operations).

## Best Practices

- Set retention high enough to cover your recovery window but low enough to avoid disk pressure.
- Use S3 storage for snapshots to offload disk space from control plane nodes.
- Monitor disk usage on control plane nodes and alert before space runs out.
- Combine local and S3 retention for defense in depth.
- Document your retention policy and review it periodically.
- Take manual snapshots before cluster upgrades regardless of automated schedule.

## Conclusion

Properly configured etcd snapshot retention balances the need for recovery options with practical storage constraints. By setting appropriate intervals and retention counts based on your RPO requirements, and combining local snapshots with S3 storage and lifecycle policies, you can maintain reliable recovery points without running into storage issues.
