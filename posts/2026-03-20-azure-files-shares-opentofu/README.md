# How to Create Azure Files Shares with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Azure Files, Storage, Infrastructure as Code, DevOps

Description: Learn how to provision Azure Files storage accounts and file shares using OpenTofu for persistent, cloud-managed SMB storage.

---

Azure Files provides fully managed cloud file shares accessible via the SMB and NFS protocols. OpenTofu lets you define storage accounts and file shares as code for consistent, reproducible deployments.

---

## Create a Storage Account

```hcl
resource "azurerm_resource_group" "storage" {
  name     = "storage-rg"
  location = "eastus"
}

resource "azurerm_storage_account" "files" {
  name                     = "myfilesstorage"
  resource_group_name      = azurerm_resource_group.storage.name
  location                 = azurerm_resource_group.storage.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
}
```

---

## Create an Azure File Share

```hcl
resource "azurerm_storage_share" "data" {
  name                 = "app-data"
  storage_account_name = azurerm_storage_account.files.name
  quota                = 100  # GB
}
```

---

## Create a Share with Specific Access Tier

```hcl
resource "azurerm_storage_share" "hot" {
  name                 = "hot-data"
  storage_account_name = azurerm_storage_account.files.name
  quota                = 500
  access_tier          = "Hot"
}
```

---

## Mount the Share from Linux

```bash
# Get the storage account key

STORAGE_KEY=$(az storage account keys list   --account-name myfilesstorage   --resource-group storage-rg   --query '[0].value' -o tsv)

# Mount via SMB
sudo mount -t cifs   //myfilesstorage.file.core.windows.net/app-data   /mnt/azure-files   -o username=myfilesstorage,password=${STORAGE_KEY},vers=3.0
```

---

## Use as a Kubernetes Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azure-files-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  azureFile:
    secretName: azure-files-secret
    shareName: app-data
    readOnly: false
```

---

## Summary

Use `azurerm_storage_account` and `azurerm_storage_share` to declare Azure Files storage in OpenTofu. Set the quota in GB and access tier to match your workload requirements. Mount shares from Linux with `cifs` or consume them as Kubernetes Persistent Volumes for shared storage across pods.
