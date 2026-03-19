# How to Roll Back a Failed Rancher Upgrade

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Upgrade

Description: Learn how to safely roll back a failed Rancher upgrade using Helm rollback and etcd snapshot restoration.

Sometimes a Rancher upgrade does not go as planned. Pods may crash, the UI may become unresponsive, or managed clusters may lose connectivity. This guide covers how to roll back a failed Rancher upgrade using Helm and, when necessary, how to restore from an etcd snapshot.

## Prerequisites

- `kubectl` and Helm 3 installed and configured
- Access to the Kubernetes cluster running Rancher
- An etcd backup taken before the upgrade (strongly recommended)

## Identifying a Failed Upgrade

Before rolling back, confirm that the upgrade has actually failed. Common signs include:

- Rancher pods stuck in `CrashLoopBackOff` or `Error` state
- The Rancher UI is inaccessible
- Helm reports the release as `FAILED`
- Managed clusters show as disconnected

Check the current state:

```bash
helm list -n cattle-system
kubectl get pods -n cattle-system
kubectl get events -n cattle-system --sort-by='.lastTimestamp' | tail -20
```

Check pod logs for errors:

```bash
kubectl logs -l app=rancher -n cattle-system --tail=50
```

## Method 1: Helm Rollback

The simplest way to roll back is using Helm. This reverts the deployment to the previous release revision.

### Step 1: List Helm Release History

```bash
helm history rancher -n cattle-system
```

This shows all revisions of the Rancher release. Identify the last successful revision number.

### Step 2: Roll Back to the Previous Revision

```bash
helm rollback rancher <REVISION_NUMBER> -n cattle-system
```

For example, to roll back to revision 5:

```bash
helm rollback rancher 5 -n cattle-system
```

### Step 3: Monitor the Rollback

```bash
kubectl rollout status deployment rancher -n cattle-system
```

Watch the pods:

```bash
kubectl get pods -n cattle-system -w
```

### Step 4: Verify the Rollback

Check that the previous version is running:

```bash
kubectl get settings server-version -o jsonpath='{.value}' -n cattle-system
helm list -n cattle-system
```

Access the Rancher UI and confirm all clusters are connected.

## Method 2: Restore from etcd Snapshot

If a Helm rollback does not resolve the issue, or if the upgrade corrupted data in etcd, you need to restore from an etcd snapshot. This is a more involved process.

### For RKE Clusters

Stop the cluster and restore the snapshot:

```bash
rke etcd snapshot-restore --config cluster.yml --name pre-upgrade-snapshot
```

Then bring the cluster back up:

```bash
rke up --config cluster.yml
```

### For RKE2 Clusters

Stop the RKE2 service on all nodes:

```bash
systemctl stop rke2-server
```

On the first server node, restore the snapshot:

```bash
rke2 server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/rke2/server/db/snapshots/pre-upgrade-snapshot
```

Start the service:

```bash
systemctl start rke2-server
```

On remaining server nodes, delete the data directory and rejoin:

```bash
rm -rf /var/lib/rancher/rke2/server/db
systemctl start rke2-server
```

### For K3s Clusters

Stop K3s on all nodes:

```bash
systemctl stop k3s
```

Restore on the first server:

```bash
k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/pre-upgrade-snapshot
```

Start K3s:

```bash
systemctl start k3s
```

## Method 3: Reinstall the Previous Version

If both Helm rollback and etcd restore fail, you can uninstall and reinstall the previous version of Rancher.

### Step 1: Save Current Values

```bash
helm get values rancher -n cattle-system -o yaml > saved-values.yaml
```

### Step 2: Uninstall Rancher

```bash
helm uninstall rancher -n cattle-system
```

### Step 3: Install the Previous Version

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --values saved-values.yaml \
  --version <PREVIOUS_VERSION>
```

### Step 4: Verify

```bash
kubectl rollout status deployment rancher -n cattle-system
kubectl get pods -n cattle-system
```

## Post-Rollback Steps

After a successful rollback, take these additional steps:

### Re-verify Managed Clusters

Check that all downstream clusters are reporting correctly:

```bash
kubectl get clusters.management.cattle.io
```

In the Rancher UI, go to each cluster and verify it shows as `Active`.

### Check Cluster Agents

Downstream cluster agents may need to reconnect after a rollback. If a cluster shows as `Unavailable`, you can re-deploy the agent by navigating to the cluster in the Rancher UI and clicking the rotate agent certificates option.

### Review Webhook Status

The Rancher webhook may be in a bad state after a rollback. Check it:

```bash
kubectl get pods -n cattle-system | grep webhook
kubectl get mutatingwebhookconfigurations | grep rancher
kubectl get validatingwebhookconfigurations | grep rancher
```

If webhooks are causing issues, you can temporarily delete them:

```bash
kubectl delete mutatingwebhookconfigurations rancher.cattle.io
kubectl delete validatingwebhookconfigurations rancher.cattle.io
```

Rancher will recreate them when it starts up properly.

### Document the Failure

Record what went wrong for future reference:

- The version you attempted to upgrade to
- Error messages from logs
- Which rollback method worked
- Any manual steps required after the rollback

## Preventing Future Upgrade Failures

- Always take an etcd snapshot before upgrading
- Test upgrades in a staging environment first
- Upgrade one minor version at a time
- Check the Rancher support matrix for Kubernetes compatibility
- Read the full release notes before upgrading
- Have a documented rollback plan ready before starting

## Conclusion

Rolling back a failed Rancher upgrade is manageable when you have prepared properly. Helm rollback is the fastest and simplest approach, while etcd snapshot restoration handles more severe failures. The key is to always have a backup before upgrading and to know the rollback procedure before you begin the upgrade process.
