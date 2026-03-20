# How to Configure Azure VM Managed Disks with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Managed Disks, Storage, Performance, Encryption, Infrastructure as Code

Description: Learn how to create and configure Azure Managed Disks with OpenTofu for VM storage, including Premium SSD, Ultra Disk, disk snapshots, and customer-managed encryption keys.

## Introduction

Azure Managed Disks are block-level storage volumes managed by Azure, eliminating the need to manage storage accounts. They come in four types: Standard HDD (archival/dev), Standard SSD (light production), Premium SSD (production workloads), and Ultra Disk (latency-sensitive databases). Managed Disks support encryption with platform-managed or customer-managed keys, incremental snapshots for backup, and disk bursting for handling traffic spikes.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials configured
- A Resource Group and optionally an existing VM for data disk attachment

## Step 1: Create Premium SSD Managed Disk

```hcl
resource "azurerm_managed_disk" "data" {
  name                 = "${var.project_name}-data-disk"
  location             = var.location
  resource_group_name  = var.resource_group_name
  storage_account_type = "Premium_LRS"  # Premium_LRS, Standard_LRS, UltraSSD_LRS
  create_option        = "Empty"
  disk_size_gb         = 512

  # Enable disk bursting for Premium SSD
  on_demand_bursting_enabled = true

  # Zone for zone-redundant deployment
  zones = ["1"]

  tags = {
    Name    = "${var.project_name}-data-disk"
    Purpose = "application-data"
  }
}
```

## Step 2: Ultra Disk for High-Performance Workloads

```hcl
resource "azurerm_managed_disk" "ultra" {
  name                 = "${var.project_name}-ultra-disk"
  location             = var.location
  resource_group_name  = var.resource_group_name
  storage_account_type = "UltraSSD_LRS"
  create_option        = "Empty"
  disk_size_gb         = 1024

  # Configure IOPS and throughput for Ultra Disk
  disk_iops_read_write = 8000    # Up to 160,000 IOPS
  disk_mbps_read_write = 512     # Up to 2,000 MB/s

  zones = ["1"]  # Ultra Disks require zone specification
}
```

## Step 3: Customer-Managed Encryption

```hcl
resource "azurerm_key_vault_key" "disk_encryption" {
  name         = "${var.project_name}-disk-key"
  key_vault_id = var.key_vault_id
  key_type     = "RSA"
  key_size     = 4096

  key_opts = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]
}

resource "azurerm_disk_encryption_set" "main" {
  name                = "${var.project_name}-disk-encryption-set"
  resource_group_name = var.resource_group_name
  location            = var.location
  key_vault_key_id    = azurerm_key_vault_key.disk_encryption.id

  identity {
    type = "SystemAssigned"
  }
}

# Grant encryption set access to Key Vault
resource "azurerm_key_vault_access_policy" "disk_encryption" {
  key_vault_id = var.key_vault_id
  tenant_id    = azurerm_disk_encryption_set.main.identity[0].tenant_id
  object_id    = azurerm_disk_encryption_set.main.identity[0].principal_id

  key_permissions = ["Get", "WrapKey", "UnwrapKey"]
}

resource "azurerm_managed_disk" "encrypted" {
  name                   = "${var.project_name}-encrypted-disk"
  location               = var.location
  resource_group_name    = var.resource_group_name
  storage_account_type   = "Premium_LRS"
  create_option          = "Empty"
  disk_size_gb           = 256
  disk_encryption_set_id = azurerm_disk_encryption_set.main.id
}
```

## Step 4: Disk Snapshot

```hcl
resource "azurerm_snapshot" "data_backup" {
  name                = "${var.project_name}-snapshot-${formatdate("YYYYMMDD", timestamp())}"
  location            = var.location
  resource_group_name = var.resource_group_name
  create_option       = "Copy"
  source_uri          = azurerm_managed_disk.data.id

  # Incremental snapshot (recommended - only stores changes since last snapshot)
  incremental_enabled = true

  tags = {
    Name      = "${var.project_name}-disk-snapshot"
    CreatedAt = timestamp()
  }

  lifecycle {
    ignore_changes = [name, tags["CreatedAt"]]
  }
}
```

## Step 5: Attach Disk to VM

```hcl
resource "azurerm_virtual_machine_data_disk_attachment" "data" {
  managed_disk_id    = azurerm_managed_disk.data.id
  virtual_machine_id = var.vm_id
  lun                = 0       # Logical unit number (0-63)
  caching            = "ReadWrite"  # None, ReadOnly, ReadWrite
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check disk performance metrics
az monitor metrics list \
  --resource <disk-id> \
  --metric "Composite Disk Read Operations/sec" "Composite Disk Write Operations/sec" \
  --interval PT1M

# Resize a disk (requires VM deallocate if OS disk)
az disk update \
  --resource-group <rg> \
  --name <disk-name> \
  --size-gb 1024
```

## Conclusion

Use `incremental_enabled = true` for snapshots in production—incremental snapshots only store block-level changes since the last snapshot, dramatically reducing snapshot costs and creation time for large disks. Premium SSD v2 and Ultra Disk require zone specification and cannot be used with VMs in Availability Sets (only Availability Zones). Set `on_demand_bursting_enabled = true` on Premium SSD disks smaller than 512 GB to handle occasional IOPS spikes without permanently paying for larger disk sizes.
