# How to Migrate Rancher Using Backup and Restore

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Backup, Restore, Disaster Recovery

Description: Learn how to migrate your Rancher management server to a new cluster using the Backup Operator's backup and restore workflow.

There are many reasons to migrate Rancher to a new cluster: hardware upgrades, cloud provider changes, Kubernetes version upgrades, or infrastructure consolidation. The Rancher Backup Operator provides a clean migration path by backing up your current Rancher state and restoring it to a new cluster. This guide walks through the entire migration process.

## Prerequisites

- Source cluster running Rancher v2.5+ with the Backup Operator
- A new target cluster ready for Rancher installation
- External storage (S3, GCS, or MinIO) accessible from both clusters
- kubectl access to both clusters
- Helm 3 installed
- The same Rancher version available for installation on the target cluster

## Step 1: Back Up the Source Rancher Installation

On the source cluster, create a backup to external storage:

```bash
kubectl create secret generic s3-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=YOUR_ACCESS_KEY \
  --from-literal=secretKey=YOUR_SECRET_KEY
```

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-migration-backup
spec:
  resourceSetName: rancher-resource-set
  storageLocation:
    s3:
      bucketName: rancher-migration
      folder: migration
      endpoint: s3.amazonaws.com
      region: us-east-1
      credentialSecretName: s3-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply and wait for completion:

```bash
kubectl apply -f migration-backup.yaml
kubectl get backups.resources.cattle.io rancher-migration-backup -w
```

Note the backup filename from the status output.

## Step 2: Prepare the Target Cluster

Set up the target cluster with the required components. Switch your kubectl context to the new cluster:

```bash
kubectl config use-context new-cluster
```

Install cert-manager (required by Rancher):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
kubectl wait --for=condition=Available -n cert-manager deployment/cert-manager --timeout=120s
```

## Step 3: Install Rancher on the Target Cluster

Install the same version of Rancher that was running on the source cluster:

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update

helm install rancher rancher-latest/rancher \
  -n cattle-system \
  --create-namespace \
  --set hostname=rancher.yourdomain.com \
  --set replicas=3 \
  --version=2.8.x
```

Wait for Rancher to become ready:

```bash
kubectl rollout status deployment rancher -n cattle-system
```

## Step 4: Install the Backup Operator on the Target Cluster

```bash
helm install rancher-backup-crd rancher-charts/rancher-backup-crd \
  -n cattle-resources-system \
  --create-namespace

helm install rancher-backup rancher-charts/rancher-backup \
  -n cattle-resources-system
```

## Step 5: Create Storage Credentials on the Target Cluster

Create the same S3 credentials secret on the target cluster:

```bash
kubectl create secret generic s3-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=YOUR_ACCESS_KEY \
  --from-literal=secretKey=YOUR_SECRET_KEY
```

If the backup was encrypted, also create the encryption secret:

```bash
kubectl create secret generic rancher-backup-encryption \
  -n cattle-resources-system \
  --from-file=encryption-provider-config.yaml=encryption-config.yaml
```

## Step 6: Scale Down Rancher Before Restore

Scale down Rancher on the target cluster to prevent conflicts during restore:

```bash
kubectl scale deployment rancher -n cattle-system --replicas=0
kubectl wait --for=delete pod -l app=rancher -n cattle-system --timeout=60s
```

## Step 7: Restore the Backup

Create the Restore resource on the target cluster:

```yaml
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: rancher-migration-restore
spec:
  backupFilename: rancher-migration-backup-2026-03-19T10-00-00Z.tar.gz
  storageLocation:
    s3:
      bucketName: rancher-migration
      folder: migration
      endpoint: s3.amazonaws.com
      region: us-east-1
      credentialSecretName: s3-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply and monitor:

```bash
kubectl apply -f migration-restore.yaml
kubectl get restores.resources.cattle.io rancher-migration-restore -w
```

Watch the operator logs for progress:

```bash
kubectl logs -n cattle-resources-system -l app.kubernetes.io/name=rancher-backup -f
```

## Step 8: Scale Rancher Back Up

Once the restore completes successfully:

```bash
kubectl scale deployment rancher -n cattle-system --replicas=3
kubectl rollout status deployment rancher -n cattle-system
```

## Step 9: Update DNS

Update your DNS records to point `rancher.yourdomain.com` to the new cluster's load balancer:

```bash
kubectl get svc -n cattle-system
```

Get the new load balancer address and update your DNS A record or CNAME.

## Step 10: Reconnect Downstream Clusters

After migration, downstream clusters need to reconnect to the new Rancher server. In most cases, this happens automatically once the DNS is updated. If clusters show as disconnected:

1. Log in to the Rancher UI on the new server.
2. Navigate to each cluster.
3. If a cluster is stuck, you may need to re-import it by running the registration command on the downstream cluster.

Get the registration command from the Rancher UI or API:

```bash
kubectl get clusterregistrationtokens.management.cattle.io -A
```

## Step 11: Verify the Migration

Confirm everything is working on the new cluster:

```bash
kubectl get clusters.management.cattle.io
kubectl get pods -n cattle-system
kubectl get nodes
```

Check the Rancher UI for:

- All clusters visible and connected
- Users and roles intact
- Catalogs and applications present
- Global settings correct
- Authentication providers configured

## Post-Migration Cleanup

Once you have verified the migration is successful:

1. Keep the source cluster running for a few days as a fallback.
2. Set up new scheduled backups on the target cluster.
3. Decommission the source cluster only after confirming everything works.
4. Remove the migration backup from S3 when no longer needed.

## Troubleshooting

### Downstream Clusters Not Reconnecting

If downstream clusters cannot connect after DNS update, restart the cattle-cluster-agent on each downstream cluster:

```bash
kubectl rollout restart deployment cattle-cluster-agent -n cattle-system
```

### Version Mismatch Errors

The Rancher version on the target must match the source. Check the Rancher version in the backup metadata and install the matching version.

### Certificate Issues

If you are using Let's Encrypt or custom certificates, ensure the TLS configuration on the target cluster matches the source.

## Conclusion

Migrating Rancher using the Backup Operator provides a reliable and repeatable process for moving your management server between clusters. By backing up to external storage, restoring on a new cluster, and updating DNS, you can complete the migration with minimal downtime and ensure all your cluster configurations, users, and settings are preserved.
