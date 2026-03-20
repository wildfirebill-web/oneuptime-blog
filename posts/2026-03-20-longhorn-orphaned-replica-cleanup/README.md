# How to Configure Longhorn Orphaned Replica Cleanup - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Orphaned Replicas, Cleanup, Kubernetes, Storage, Disk Space, SUSE Rancher

Description: Learn how to configure Longhorn's automatic orphaned replica cleanup to reclaim disk space consumed by replicas that are no longer associated with any volume.

---

Orphaned replicas are Longhorn replica data directories on nodes that are no longer associated with any Longhorn volume. They accumulate over time from deleted volumes, failed replica scheduling, or cluster migrations and consume disk space unnecessarily.

---

## What Creates Orphaned Replicas

- Volume deleted without proper Longhorn cleanup
- Node failure during replica scheduling
- Manual deletion of Longhorn volume objects without deleting the PVC
- Cluster restored from backup with stale replica directories

---

## Step 1: Enable Automatic Orphaned Replica Cleanup

Configure Longhorn to automatically detect and clean up orphaned replicas:

```bash
# Enable orphaned replica auto cleanup in Longhorn settings

kubectl patch setting.longhorn.io orphan-auto-deletion \
  -n longhorn-system \
  --type merge \
  -p '{"value":"true"}'
```

---

## Step 2: Manually Identify Orphaned Data

View orphaned data through the Longhorn UI or via kubectl:

```bash
# List detected orphaned resources
kubectl get lhorphan -n longhorn-system

# Get details of a specific orphaned replica
kubectl describe lhorphan <orphan-name> -n longhorn-system
```

---

## Step 3: Manually Delete Specific Orphans

```bash
# Delete a specific orphaned replica directory
kubectl delete lhorphan <orphan-name> -n longhorn-system

# Delete all orphaned replicas (use with caution!)
kubectl delete lhorphan -n longhorn-system --all
```

---

## Step 4: Verify Disk Space Is Reclaimed

After cleanup, confirm disk space was recovered:

```bash
# Check node disk usage before and after
kubectl get lhnode -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,USED:.status.diskStatus'

# On the node directly
df -h /var/lib/longhorn
```

---

## Step 5: Prevent Orphaned Replica Accumulation

Configure settings to automatically clean up replicas when volumes are deleted:

```bash
# Auto-delete orphaned replicas after volumes are deleted
kubectl patch setting.longhorn.io remove-snapshots-during-filesystem-trim \
  -n longhorn-system \
  --type merge \
  -p '{"value":"enabled"}'
```

---

## Step 6: Scheduled Orphan Detection

Set up a CronJob to periodically check for orphans and report them:

```yaml
# orphan-report-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: longhorn-orphan-report
  namespace: longhorn-system
spec:
  schedule: "0 6 * * 0"   # Every Sunday at 6 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: longhorn-service-account
          containers:
            - name: reporter
              image: bitnami/kubectl:latest
              command:
                - sh
                - -c
                - |
                  COUNT=$(kubectl get lhorphan -n longhorn-system --no-headers | wc -l)
                  echo "Orphaned Longhorn replicas detected: $COUNT"
                  if [ $COUNT -gt 0 ]; then
                    kubectl get lhorphan -n longhorn-system
                  fi
          restartPolicy: OnFailure
```

---

## Best Practices

- Enable `orphan-auto-deletion` on all production clusters to prevent disk space accumulation.
- Review orphans before mass deletion - occasionally an orphan may be a recoverable replica from a volume that still has data you need.
- Run orphan cleanup after any cluster restore operation to clear stale replica data.
- Monitor disk usage trends - rapidly increasing orphan count can indicate a misconfigured or buggy workload deleting and recreating PVCs frequently.
