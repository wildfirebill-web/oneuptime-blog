# How to Back Up Rancher to Azure Blob Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Backup, Azure Blob

Description: Learn how to configure the Rancher Backup Operator to store backups in Azure Blob Storage for durable, cloud-native backup storage.

If your organization runs on Azure, storing Rancher backups in Azure Blob Storage keeps your backup infrastructure within the same cloud ecosystem. The Rancher Backup Operator supports S3-compatible storage, and with Azure Blob's S3 compatibility layer or a proxy approach, you can store your backups there. This guide covers the complete setup.

## Prerequisites

- Rancher v2.5 or later with the Backup Operator installed
- An Azure account with a Storage Account
- Azure CLI installed and configured
- kubectl access with cluster admin privileges

## Step 1: Create an Azure Storage Account

If you do not already have a storage account, create one:

```bash
az group create --name rancher-backup-rg --location eastus

az storage account create \
  --name rancherbackupstorage \
  --resource-group rancher-backup-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot
```

## Step 2: Create a Blob Container

Create a container within the storage account to hold backups:

```bash
az storage container create \
  --name rancher-backups \
  --account-name rancherbackupstorage
```

## Step 3: Get Storage Account Credentials

Retrieve the storage account access key:

```bash
az storage account keys list \
  --account-name rancherbackupstorage \
  --resource-group rancher-backup-rg \
  --query '[0].value' -o tsv
```

Save this key for the next step.

## Step 4: Use MinIO Gateway for Azure

Since the Rancher Backup Operator natively supports S3-compatible endpoints, you can deploy a MinIO gateway that translates S3 API calls to Azure Blob Storage. Deploy MinIO gateway in your cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio-azure-gateway
  namespace: cattle-resources-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio-azure-gateway
  template:
    metadata:
      labels:
        app: minio-azure-gateway
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args:
        - gateway
        - azure
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              name: azure-storage-creds
              key: accountName
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: azure-storage-creds
              key: accountKey
        ports:
        - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: minio-azure-gateway
  namespace: cattle-resources-system
spec:
  selector:
    app: minio-azure-gateway
  ports:
  - port: 9000
    targetPort: 9000
```

Create the Azure storage credentials secret:

```bash
kubectl create secret generic azure-storage-creds \
  -n cattle-resources-system \
  --from-literal=accountName=rancherbackupstorage \
  --from-literal=accountKey=YOUR_AZURE_STORAGE_KEY
```

Apply the MinIO gateway deployment:

```bash
kubectl apply -f minio-azure-gateway.yaml
```

## Step 5: Create S3 Credentials for the Backup Operator

Create a secret with the MinIO gateway credentials (which are the Azure storage account name and key):

```bash
kubectl create secret generic azure-s3-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=rancherbackupstorage \
  --from-literal=secretKey=YOUR_AZURE_STORAGE_KEY
```

## Step 6: Create a Backup Targeting Azure Blob

Create the Backup resource pointing to the MinIO gateway:

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-backup-azure
spec:
  resourceSetName: rancher-resource-set
  retentionCount: 10
  storageLocation:
    s3:
      bucketName: rancher-backups
      endpoint: minio-azure-gateway.cattle-resources-system.svc:9000
      insecureTLSSkipVerify: true
      credentialSecretName: azure-s3-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply the backup:

```bash
kubectl apply -f backup-azure.yaml
```

## Step 7: Set Up Scheduled Backups

For automated daily backups to Azure:

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-daily-azure-backup
spec:
  resourceSetName: rancher-resource-set
  retentionCount: 30
  schedule: "0 2 * * *"
  storageLocation:
    s3:
      bucketName: rancher-backups
      folder: daily
      endpoint: minio-azure-gateway.cattle-resources-system.svc:9000
      insecureTLSSkipVerify: true
      credentialSecretName: azure-s3-creds
      credentialSecretNamespace: cattle-resources-system
```

## Step 8: Verify Backups in Azure

Check the backup status:

```bash
kubectl get backups.resources.cattle.io rancher-backup-azure -o yaml
```

Verify the blob exists in Azure Storage:

```bash
az storage blob list \
  --container-name rancher-backups \
  --account-name rancherbackupstorage \
  --output table
```

## Step 9: Configure Blob Lifecycle Management

Set up lifecycle rules to manage storage costs:

```bash
az storage account management-policy create \
  --account-name rancherbackupstorage \
  --resource-group rancher-backup-rg \
  --policy '{
    "rules": [
      {
        "enabled": true,
        "name": "archiveOldBackups",
        "type": "Lifecycle",
        "definition": {
          "actions": {
            "baseBlob": {
              "tierToCool": {"daysAfterModificationGreaterThan": 30},
              "tierToArchive": {"daysAfterModificationGreaterThan": 90}
            }
          },
          "filters": {
            "blobTypes": ["blockBlob"],
            "prefixMatch": ["rancher-backups/"]
          }
        }
      }
    ]
  }'
```

## Troubleshooting

### Gateway Connection Issues

Verify the MinIO gateway pod is running:

```bash
kubectl get pods -n cattle-resources-system -l app=minio-azure-gateway
kubectl logs -n cattle-resources-system -l app=minio-azure-gateway
```

### Authentication Failures

Double-check that the storage account name and key are correct in the secrets. Retrieve the key again if needed:

```bash
az storage account keys list \
  --account-name rancherbackupstorage \
  --resource-group rancher-backup-rg
```

### Blob Not Appearing

Ensure the container name in the Backup resource matches the Azure Blob container name exactly.

## Conclusion

By combining the Rancher Backup Operator with a MinIO gateway to Azure Blob Storage, you can store your Rancher backups in Azure's durable cloud storage. This approach integrates well with Azure-native lifecycle management and access control, giving you a reliable backup strategy within your existing Azure infrastructure.
