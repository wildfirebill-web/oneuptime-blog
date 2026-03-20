# How to Configure Azure Files with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure Files, SMB, NFS, Storage, Infrastructure as Code

Description: Learn how to configure Azure Files with OpenTofu - creating file shares, configuring SMB and NFS protocols, private endpoints, Azure File Sync, and mounting shares on Azure VMs.

## Introduction

Azure Files provides fully managed cloud file shares accessible via SMB or NFS protocols. They serve as lift-and-shift replacements for on-premises file servers, shared application configuration stores, and persistent storage for containers. OpenTofu manages the storage account, file shares, private endpoints, and access configuration.

## Storage Account for Azure Files

```hcl
resource "azurerm_storage_account" "files" {
  name                = "${var.environment}filestorage"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location

  account_tier             = "Premium"    # Premium for low-latency (<1ms)
  account_replication_type = "LRS"        # or ZRS for zone redundancy
  account_kind             = "FileStorage" # Required for Premium Azure Files

  https_traffic_only_enabled      = true
  min_tls_version                 = "TLS1_2"
  allow_nested_items_to_be_public = false

  # SMB security settings
  share_properties {
    smb {
      versions                        = ["SMB3.1.1"]
      authentication_types            = ["Kerberos"]
      kerberos_ticket_encryption_type = ["AES-256"]
      channel_encryption_type         = ["AES-256-GCM"]
    }

    retention_policy {
      days = 7
    }
  }

  tags = { Environment = var.environment }
}
```

## File Shares

```hcl
# SMB file share (Windows compatible)

resource "azurerm_storage_share" "app_data" {
  name                 = "app-data"
  storage_account_name = azurerm_storage_account.files.name
  quota                = 100  # GB - soft quota (Premium)
  enabled_protocol     = "SMB"

  acl {
    id = "app-read"

    access_policy {
      permissions = "r"
      start       = "2024-01-01T00:00:00Z"
      expiry      = "2030-01-01T00:00:00Z"
    }
  }
}

# NFS file share (Linux compatible, requires Premium tier)
resource "azurerm_storage_share" "linux_data" {
  name                 = "linux-data"
  storage_account_name = azurerm_storage_account.files.name
  quota                = 500
  enabled_protocol     = "NFS"
}
```

## Private Endpoint for Azure Files

```hcl
resource "azurerm_private_endpoint" "files" {
  name                = "${var.environment}-files-pe"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  subnet_id           = azurerm_subnet.private.id

  private_service_connection {
    name                           = "files-connection"
    private_connection_resource_id = azurerm_storage_account.files.id
    subresource_names              = ["file"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "files-dns-group"
    private_dns_zone_ids = [azurerm_private_dns_zone.files.id]
  }
}

resource "azurerm_private_dns_zone" "files" {
  name                = "privatelink.file.core.windows.net"
  resource_group_name = azurerm_resource_group.app.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "files" {
  name                  = "files-dns-link"
  resource_group_name   = azurerm_resource_group.app.name
  private_dns_zone_name = azurerm_private_dns_zone.files.name
  virtual_network_id    = azurerm_virtual_network.app.id
}

# Restrict access to private endpoint only
resource "azurerm_storage_account_network_rules" "files" {
  storage_account_id = azurerm_storage_account.files.id
  default_action     = "Deny"
  bypass             = ["AzureServices"]
}
```

## RBAC Access to File Share

```hcl
# Grant VM managed identity access to the file share
resource "azurerm_role_assignment" "files_contributor" {
  scope                = azurerm_storage_account.files.id
  role_definition_name = "Storage File Data SMB Share Contributor"
  principal_id         = azurerm_linux_virtual_machine.app.identity[0].principal_id
}

resource "azurerm_role_assignment" "files_reader" {
  scope                = azurerm_storage_account.files.id
  role_definition_name = "Storage File Data SMB Share Reader"
  principal_id         = azurerm_user_assigned_identity.readonly.principal_id
}
```

## Azure File Sync Configuration

```hcl
# Azure File Sync replicates on-premises file servers with Azure Files
resource "azurerm_storage_sync" "main" {
  name                = "${var.environment}-file-sync"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location

  tags = { Environment = var.environment }
}

resource "azurerm_storage_sync_group" "app" {
  name            = "app-sync-group"
  storage_sync_id = azurerm_storage_sync.main.id
}

resource "azurerm_storage_sync_cloud_endpoint" "app" {
  name                  = "app-cloud-endpoint"
  storage_sync_group_id = azurerm_storage_sync_group.app.id
  file_share_name       = azurerm_storage_share.app_data.name
  storage_account_id    = azurerm_storage_account.files.id
}
```

## VM Custom Data for SMB Mount

```hcl
resource "azurerm_linux_virtual_machine" "app" {
  name                = "${var.environment}-app-vm"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  size                = "Standard_D2s_v3"
  admin_username      = "azureuser"

  custom_data = base64encode(<<-EOF
    #!/bin/bash
    apt-get install -y cifs-utils

    # Mount SMB share using storage account key
    STORAGE_ACCOUNT="${azurerm_storage_account.files.name}"
    SHARE_NAME="${azurerm_storage_share.app_data.name}"
    STORAGE_KEY="${azurerm_storage_account.files.primary_access_key}"

    mkdir -p /mnt/appdata

    echo "//${azurerm_storage_account.files.name}.file.core.windows.net/$SHARE_NAME /mnt/appdata cifs vers=3.0,username=$STORAGE_ACCOUNT,password=$STORAGE_KEY,dir_mode=0755,file_mode=0644,serverino 0 0" >> /etc/fstab

    mount -a
  EOF
  )

  # ... other VM configuration
}
```

## Outputs

```hcl
output "smb_connection_string" {
  value       = "\\\\${azurerm_storage_account.files.name}.file.core.windows.net\\${azurerm_storage_share.app_data.name}"
  description = "SMB UNC path for Windows clients"
}

output "nfs_mount_path" {
  value       = "${azurerm_storage_account.files.name}.file.core.windows.net:/${azurerm_storage_account.files.name}/${azurerm_storage_share.linux_data.name}"
  description = "NFS mount path for Linux clients"
}
```

## Conclusion

Azure Files with OpenTofu provides managed SMB and NFS file shares without server administration. Use Premium tier (`account_kind = "FileStorage"`) for sub-millisecond latency required by workloads migrating from local NAS. Deploy private endpoints so all file access stays within the VNet - never traversing the internet. Use Azure AD Kerberos authentication (`authentication_types = ["Kerberos"]`) for user-level access control instead of storage account keys, enabling per-user audit trails in Azure Monitor.
