# How to Migrate Rancher from Docker Install to Kubernetes Install

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Migration

Description: Step-by-step guide to migrating a single-node Docker Rancher installation to a production-grade Kubernetes-based deployment.

Running Rancher as a single Docker container is fine for testing and small environments, but production workloads need the resilience and scalability of a Kubernetes-based high-availability deployment. This guide walks you through migrating your Rancher installation from Docker to Kubernetes without losing your managed clusters and configurations.

## Prerequisites

- An existing Rancher instance running as a Docker container
- A target Kubernetes cluster ready for the Rancher Helm installation (RKE2, K3s, or RKE)
- Helm 3 installed
- `kubectl` configured for the target cluster
- `docker` CLI access to the source Rancher container

## Step 1: Document Your Current Setup

Before starting the migration, record your current configuration:

```bash
# Check the Rancher version
docker exec rancher kubectl get settings server-version -o jsonpath='{.value}'

# Note the hostname
docker exec rancher kubectl get settings server-url -o jsonpath='{.value}'

# List managed clusters
docker exec rancher kubectl get clusters.management.cattle.io -o custom-columns=NAME:.metadata.name,DISPLAY:.spec.displayName
```

Save this information for reference during the migration.

## Step 2: Create a Backup of the Docker Rancher

Stop the Rancher container and create a backup:

```bash
# Stop the container
docker stop rancher

# Create a backup of the container's data volume
docker create --volumes-from rancher --name rancher-backup busybox true
docker cp rancher-backup:/var/lib/rancher /tmp/rancher-backup

# Alternatively, if using a bind mount
cp -r /opt/rancher /tmp/rancher-backup
```

You should also export the etcd data from inside the container:

```bash
docker start rancher
docker exec rancher sh -c "ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  --cert=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt \
  --key=/var/lib/rancher/k3s/server/tls/etcd/server-client.key"

docker cp rancher:/tmp/etcd-snapshot.db /tmp/etcd-snapshot.db
```

## Step 3: Set Up the Target Kubernetes Cluster

If you do not already have a target cluster, create one using RKE2:

```bash
# Install RKE2 on the first server node
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server
systemctl start rke2-server
```

Configure kubectl:

```bash
mkdir -p ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
chmod 600 ~/.kube/config
```

For a production setup, add at least two more server nodes for high availability.

## Step 4: Install cert-manager on the Target Cluster

Rancher requires cert-manager for TLS certificate management:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.4
```

Verify cert-manager is running:

```bash
kubectl get pods -n cert-manager
```

## Step 5: Install Rancher on the Target Cluster

Install the same version of Rancher that is running on your Docker instance:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.yourdomain.com \
  --set replicas=3 \
  --set bootstrapPassword=admin \
  --version <SAME_VERSION_AS_DOCKER>
```

Wait for Rancher to be ready:

```bash
kubectl rollout status deployment rancher -n cattle-system
```

## Step 6: Migrate Data Using the Rancher Backup Operator

The recommended approach for migrating data is using the Rancher Backup and Restore operator.

### Install Backup Operator on the Docker Rancher

Access the Docker Rancher UI and install the Backup and Restore operator from the Rancher Marketplace under the local cluster.

Create a backup from the UI or with kubectl from inside the container:

```bash
docker exec rancher kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: migration-backup
spec:
  resourceSetName: rancher-resource-set
  storageLocation:
    s3:
      bucketName: rancher-backups
      endpoint: s3.amazonaws.com
      region: us-east-1
      credentialSecretName: s3-creds
      credentialSecretNamespace: cattle-resources-system
EOF
```

Alternatively, create a local backup and copy it out:

```bash
docker exec rancher kubectl apply -f - <<EOF
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: migration-backup
spec:
  resourceSetName: rancher-resource-set
EOF
```

### Install Backup Operator on the Target Cluster

```bash
helm repo add rancher-charts https://charts.rancher.io
helm install rancher-backup-crd rancher-charts/rancher-backup-crd -n cattle-resources-system --create-namespace
helm install rancher-backup rancher-charts/rancher-backup -n cattle-resources-system
```

### Restore the Backup on the Target Cluster

Transfer the backup file to a location accessible by the target cluster, then create a Restore resource:

```yaml
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: migration-restore
spec:
  backupFilename: migration-backup-<timestamp>.tar.gz
  storageLocation:
    s3:
      bucketName: rancher-backups
      endpoint: s3.amazonaws.com
      region: us-east-1
      credentialSecretName: s3-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply it:

```bash
kubectl apply -f restore.yaml
```

Monitor the restore progress:

```bash
kubectl get restores -w
```

## Step 7: Update DNS

Point your Rancher hostname DNS record to the new Kubernetes cluster's load balancer or ingress IP:

```bash
# Get the ingress IP
kubectl get ingress -n cattle-system
```

Update your DNS records to point to the new IP address.

## Step 8: Verify the Migration

Log in to the Rancher UI on the new cluster and check:

- All managed clusters appear and show as `Active`
- Users, roles, and permissions are intact
- Projects and namespaces are correctly assigned
- Catalogs and apps are available

## Step 9: Decommission the Docker Instance

Once you have verified everything works on the new Kubernetes-based installation, stop and remove the Docker container:

```bash
docker stop rancher
docker rm rancher
```

Keep the backup data for at least 30 days in case you need to reference it.

## Troubleshooting

- **Clusters show as Unavailable**: The cluster agents may need to reconnect. Wait a few minutes, or re-import the clusters from the new Rancher.
- **Certificate issues**: Make sure TLS certificates match the hostname and are properly configured.
- **Version mismatch**: The target Rancher version must match the source version before restoring backups.

## Conclusion

Migrating Rancher from a Docker installation to Kubernetes gives you high availability, easier upgrades, and better scalability. By using the Rancher Backup and Restore operator, you can transfer your configurations, clusters, and settings without manually recreating everything. Plan the migration carefully, test in a staging environment, and keep your backups until the new installation is confirmed stable.
