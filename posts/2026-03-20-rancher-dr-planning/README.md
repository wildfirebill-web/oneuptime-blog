# How to Plan Disaster Recovery for Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: rancher, disaster-recovery, kubernetes, backup, planning

Description: A comprehensive guide to planning disaster recovery strategies for Rancher environments to minimize downtime and data loss.

## Introduction

Disaster recovery (DR) planning is essential for any production Rancher deployment. A well-designed DR plan ensures that your Kubernetes infrastructure can recover quickly from failures ranging from individual node outages to complete data center disasters. This guide walks through the key components of a comprehensive DR plan for Rancher.

## Understanding Recovery Objectives

Before building your DR plan, define your recovery objectives:

- **Recovery Time Objective (RTO)**: The maximum acceptable time to restore Rancher services after a disaster. For most production environments, this should be under 4 hours.
- **Recovery Point Objective (RPO)**: The maximum acceptable data loss measured in time. For Rancher, this typically means how frequently you back up etcd.

## Key Components to Protect

### Rancher Management Server
- etcd database (contains all cluster state)
- Rancher configuration and settings
- TLS certificates and keys
- kubeconfig files

### Downstream Clusters
- Cluster configurations
- Persistent volumes and their data
- Namespace-level resources
- Secrets and ConfigMaps

## DR Planning Steps

### Step 1: Inventory Your Infrastructure

```bash
# List all managed clusters
rancher cluster ls

# Export cluster configurations
for cluster in $(rancher cluster ls --format "{{.ID}}"); do
  rancher cluster export $cluster > backup-${cluster}.yaml
done
```

### Step 2: Define Backup Schedules

```yaml
# recurring-backup-schedule.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: rancher-etcd-backup
  namespace: cattle-system
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: rancher/backup-restore-operator:latest
            command:
            - /bin/sh
            - -c
            - "etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db"
          restartPolicy: OnFailure
```

### Step 3: Choose Your DR Architecture

| Architecture | RTO | RPO | Cost | Complexity |
|---|---|---|---|---|
| Cold Standby | 4-8 hours | 1-6 hours | Low | Low |
| Warm Standby | 1-4 hours | 15-60 min | Medium | Medium |
| Hot Standby | < 1 hour | < 15 min | High | High |

### Step 4: Document Recovery Procedures

Create runbooks for each failure scenario:

```markdown
## Runbook: Rancher Server Total Failure

### Prerequisites
- Access to backup storage (S3/NFS)
- New server matching original specs
- Valid backup files

### Steps
1. Provision new server
2. Install Docker/Kubernetes
3. Restore etcd snapshot
4. Restore Rancher deployment
5. Verify cluster connectivity
6. Update DNS records
7. Validate all downstream clusters
```

### Step 5: Establish Communication Plans

- Define incident commander role
- Create escalation matrix
- Set up out-of-band communication channels
- Document external dependencies (DNS, load balancers, storage)

## Backup Configuration with Rancher Backup Operator

```yaml
# backup-resource.yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-daily-backup
spec:
  storageLocation:
    s3:
      bucketName: rancher-dr-backups
      folder: daily
      region: us-east-1
      endpoint: s3.amazonaws.com
  schedule: "0 2 * * *"   # Daily at 2 AM
  retentionCount: 14       # Keep 14 backups
  encryptionConfigSecretName: backup-encryption-key
```

## Testing Your DR Plan

A DR plan that hasn't been tested is just a document. Schedule regular DR drills:

```bash
# Quarterly DR drill checklist
echo "=== DR Drill Checklist ==="
echo "[ ] Backup files accessible from secondary location"
echo "[ ] Restoration procedure documented and up-to-date"
echo "[ ] Team familiar with recovery steps"
echo "[ ] RTO/RPO targets validated"
echo "[ ] Communication plan tested"
echo "[ ] DNS failover configured"
```

## Monitoring and Alerting for DR Readiness

```yaml
# PrometheusRule for backup monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rancher-backup-alerts
  namespace: cattle-system
spec:
  groups:
  - name: rancher-backup
    rules:
    - alert: RancherBackupFailed
      expr: rancher_backup_last_success_timestamp < (time() - 86400)
      for: 1h
      labels:
        severity: critical
      annotations:
        summary: "Rancher backup has not succeeded in 24 hours"
```

## Conclusion

A solid DR plan for Rancher involves clear objectives, regular backups, tested recovery procedures, and well-defined communication channels. Start with defining your RTO and RPO, then work backwards to implement the backup and recovery mechanisms that meet those targets. Regularly test your plan to ensure it works when you need it most.
