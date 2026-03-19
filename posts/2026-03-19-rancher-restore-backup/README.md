# How to Restore Rancher from a Backup

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Restore, Backup

Description: Learn how to restore your Rancher management server from a backup created by the Rancher Backup Operator.

When disaster strikes or you need to recover from a failed upgrade, restoring Rancher from a backup is essential. The Rancher Backup Operator provides a Restore custom resource that lets you bring your Rancher management server back to a known good state. This guide walks you through the full restore process.

## Prerequisites

- A backup file created by the Rancher Backup Operator (`.tar.gz` format)
- The Rancher Backup Operator installed on the target cluster
- kubectl access to the local (upstream) cluster with admin privileges
- Helm 3 installed on your workstation
- The same Rancher version as when the backup was taken (or compatible version)

## Step 1: Prepare the Target Cluster

If you are restoring to a fresh cluster, first install Rancher and the Backup Operator. If you are restoring to the same cluster, ensure the Backup Operator is still running.

Verify the operator is available:

```bash
kubectl get pods -n cattle-resources-system
```

If the operator is not installed, install it:

```bash
helm repo add rancher-charts https://charts.rancher.io
helm repo update

helm install rancher-backup-crd rancher-charts/rancher-backup-crd \
  -n cattle-resources-system \
  --create-namespace

helm install rancher-backup rancher-charts/rancher-backup \
  -n cattle-resources-system
```

## Step 2: Make the Backup File Available

The backup file must be accessible to the Backup Operator. Depending on where your backup is stored, you have several options.

### Option A: Local Storage

If the backup file is stored locally, copy it to the operator pod:

```bash
BACKUP_POD=$(kubectl get pods -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup -o jsonpath='{.items[0].metadata.name}')

kubectl cp rancher-backup-1-2026-03-19T10-00-00Z.tar.gz \
  cattle-resources-system/$BACKUP_POD:/var/lib/backups/
```

### Option B: S3 Storage

If your backup is in S3, create the credentials secret:

```bash
kubectl create secret generic s3-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=YOUR_ACCESS_KEY \
  --from-literal=secretKey=YOUR_SECRET_KEY
```

## Step 3: Scale Down Rancher

Before restoring, scale down the Rancher deployment to prevent conflicts:

```bash
kubectl scale deployment rancher -n cattle-system --replicas=0
```

Wait for the pods to terminate:

```bash
kubectl get pods -n cattle-system -w
```

## Step 4: Create the Restore Resource

Create a Restore custom resource. Save the following as `restore.yaml`:

### For Local Backup:

```yaml
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: rancher-restore-1
spec:
  backupFilename: rancher-backup-1-2026-03-19T10-00-00Z.tar.gz
```

### For S3 Backup:

```yaml
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: rancher-restore-1
spec:
  backupFilename: rancher-backup-1-2026-03-19T10-00-00Z.tar.gz
  storageLocation:
    s3:
      bucketName: rancher-backups
      endpoint: s3.amazonaws.com
      region: us-east-1
      credentialSecretName: s3-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply the restore resource:

```bash
kubectl apply -f restore.yaml
```

## Step 5: Monitor the Restore Process

Watch the restore progress:

```bash
kubectl get restores.resources.cattle.io rancher-restore-1 -o yaml
```

Check the conditions in the status section:

```yaml
status:
  conditions:
  - lastUpdateTime: "2026-03-19T11:00:00Z"
    status: "True"
    type: Ready
  restoreCompletionTs: "2026-03-19T11:00:30Z"
```

You can also watch the operator logs for detailed progress:

```bash
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup -f
```

## Step 6: Scale Rancher Back Up

Once the restore completes successfully, scale Rancher back up:

```bash
kubectl scale deployment rancher -n cattle-system --replicas=3
```

Wait for the pods to become ready:

```bash
kubectl rollout status deployment rancher -n cattle-system
```

## Step 7: Verify the Restore

Log in to the Rancher UI and verify that:

- All clusters are visible and in the expected state
- Users and roles are intact
- Global settings match your previous configuration
- Catalogs and apps are present
- Authentication providers are configured correctly

From the command line, check that Rancher is healthy:

```bash
kubectl get pods -n cattle-system
kubectl get clusters.management.cattle.io
```

## Troubleshooting Common Issues

### Restore Fails with Version Mismatch

Ensure the Rancher version on the target cluster matches the version used when the backup was created. Cross-version restores may not be supported.

### Resources Already Exist

If resources already exist on the cluster, the restore process will attempt to update them. If conflicts occur, you may need to delete the existing resources first:

```bash
kubectl delete clusters.management.cattle.io --all
```

Use caution when deleting resources and only do so if you are certain the backup contains the correct state.

### Operator Pod Crashes

Check the operator logs for errors:

```bash
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup --previous
```

Ensure sufficient memory and CPU resources are allocated to the operator pod.

## Conclusion

Restoring Rancher from a backup is a straightforward process when you have a valid backup file and the Backup Operator installed. By following these steps, you can recover your Rancher management server quickly and minimize downtime. Always test your restore process in a staging environment before relying on it for production recovery.
