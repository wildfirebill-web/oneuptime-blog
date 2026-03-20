# How to Migrate Rancher to a New Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Migration, Cluster Management

Description: A complete guide to migrating your Rancher management server from one Kubernetes cluster to another using backup and restore.

There are many reasons to migrate Rancher to a new cluster: hardware refresh, moving to a different cloud provider, upgrading the underlying Kubernetes distribution, or consolidating infrastructure. This guide walks you through the complete process of migrating Rancher and all its managed cluster data to a new Kubernetes cluster.

## Prerequisites

- Access to both the source and target Kubernetes clusters
- Helm 3 and kubectl installed
- The Rancher Backup and Restore operator (or willingness to install it)
- S3-compatible storage for backup transfer (recommended) or local file transfer capability
- DNS control for the Rancher hostname

## Step 1: Prepare the Source Cluster

### Document the Current Configuration

Record the current setup for reference:

```bash
# Current Rancher version

kubectl get settings server-version -o jsonpath='{.value}' -n cattle-system

# Current Helm values
helm get values rancher -n cattle-system -o yaml > source-values.yaml

# List of managed clusters
kubectl get clusters.management.cattle.io -o custom-columns=NAME:.metadata.name,DISPLAY:.spec.displayName
```

### Install the Backup Operator

If not already installed, add the Rancher Backup and Restore operator:

```bash
helm repo add rancher-charts https://charts.rancher.io
helm repo update

helm install rancher-backup-crd rancher-charts/rancher-backup-crd \
  -n cattle-resources-system --create-namespace

helm install rancher-backup rancher-charts/rancher-backup \
  -n cattle-resources-system
```

### Create S3 Credentials Secret

```bash
kubectl create secret generic s3-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=YOUR_ACCESS_KEY \
  --from-literal=secretKey=YOUR_SECRET_KEY
```

### Take the Backup

Create a backup resource:

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: cluster-migration-backup
spec:
  resourceSetName: rancher-resource-set
  storageLocation:
    s3:
      bucketName: rancher-migration
      endpoint: s3.amazonaws.com
      region: us-east-1
      credentialSecretName: s3-creds
      credentialSecretNamespace: cattle-resources-system
```

```bash
kubectl apply -f backup.yaml
```

Monitor until complete:

```bash
kubectl get backups cluster-migration-backup -w
```

## Step 2: Set Up the Target Cluster

### Provision the New Cluster

Set up a Kubernetes cluster using your preferred distribution. For example, with RKE2:

```bash
# On each server node
curl -sfL https://get.rke2.io | sh -

# On the first server node, configure and start
cat > /etc/rancher/rke2/config.yaml <<EOF
tls-san:
  - rancher.yourdomain.com
  - <LOAD_BALANCER_IP>
EOF

systemctl enable rke2-server
systemctl start rke2-server
```

Add additional server and worker nodes as needed for high availability.

### Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.4

kubectl get pods -n cert-manager
```

### Install Rancher

Install the same version of Rancher as the source cluster:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --values source-values.yaml \
  --version <SAME_VERSION>
```

Wait for it to be ready:

```bash
kubectl rollout status deployment rancher -n cattle-system
```

## Step 3: Restore the Backup on the Target Cluster

### Install the Backup Operator

```bash
helm install rancher-backup-crd rancher-charts/rancher-backup-crd \
  -n cattle-resources-system --create-namespace

helm install rancher-backup rancher-charts/rancher-backup \
  -n cattle-resources-system
```

### Create S3 Credentials

```bash
kubectl create secret generic s3-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=YOUR_ACCESS_KEY \
  --from-literal=secretKey=YOUR_SECRET_KEY
```

### Perform the Restore

```yaml
apiVersion: resources.cattle.io/v1
kind: Restore
metadata:
  name: cluster-migration-restore
spec:
  backupFilename: cluster-migration-backup-<timestamp>.tar.gz
  storageLocation:
    s3:
      bucketName: rancher-migration
      endpoint: s3.amazonaws.com
      region: us-east-1
      credentialSecretName: s3-creds
      credentialSecretNamespace: cattle-resources-system
```

```bash
kubectl apply -f restore.yaml
kubectl get restores cluster-migration-restore -w
```

After the restore completes, restart Rancher:

```bash
kubectl rollout restart deployment rancher -n cattle-system
kubectl rollout status deployment rancher -n cattle-system
```

## Step 4: Switch DNS

Update your DNS records to point the Rancher hostname to the new cluster's load balancer IP:

```bash
kubectl get svc -n cattle-system
kubectl get ingress -n cattle-system
```

Use your DNS provider to update the A record. If you use a short TTL, this transition will be faster.

## Step 5: Verify the Migration

Log in to the Rancher UI and perform these checks:

- All managed clusters appear in the cluster list
- Clusters show as `Active` (give them a few minutes to reconnect)
- Users, roles, and RBAC settings are preserved
- Projects and namespaces are intact
- Global settings match your previous configuration
- Catalogs and installed apps are visible

Check cluster agent connectivity:

```bash
kubectl get clusters.management.cattle.io -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions
```

## Step 6: Re-establish Agent Connections

Some downstream clusters may need their agents to reconnect. If a cluster remains `Unavailable`, you can rotate the agent certificates from the Rancher UI or re-register the cluster.

For clusters that need re-registration:

1. Go to the cluster in the Rancher UI
2. Navigate to cluster management options
3. Select the option to rotate certificates or regenerate the registration command
4. Apply the new agent YAML on the downstream cluster

## Step 7: Decommission the Source Cluster

Once all clusters are confirmed active and working on the new installation:

1. Remove the old DNS records
2. Uninstall Rancher from the source cluster:

```bash
helm uninstall rancher -n cattle-system
```

3. Keep the S3 backup for at least 30 days
4. Decommission the source cluster infrastructure as needed

## Troubleshooting

- **Restore fails**: Ensure the Rancher version on the target matches the source exactly.
- **Clusters stuck in Unavailable**: Check agent logs on downstream clusters with `kubectl logs -n cattle-system -l app=cattle-cluster-agent`.
- **Authentication issues**: If using external auth (LDAP, AD, SAML), verify the target cluster can reach the auth endpoints.
- **Certificate errors**: Regenerate certificates if the domain or load balancer IP changed.

## Conclusion

Migrating Rancher to a new cluster is a structured process: back up the source, set up the target, restore the data, switch DNS, and verify. The Rancher Backup and Restore operator makes this process reliable and repeatable. Always test the migration in a non-production environment first, and keep backups available until you are confident the new installation is stable.
