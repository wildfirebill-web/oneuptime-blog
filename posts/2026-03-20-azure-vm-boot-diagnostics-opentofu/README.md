# How to Set Up Azure VM Boot Diagnostics with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Boot Diagnostics, Serial Console, Troubleshooting, VMs, Infrastructure as Code

Description: Learn how to configure Azure VM Boot Diagnostics with OpenTofu to capture serial console output and screenshots for troubleshooting VM startup issues and enabling Azure Serial Console access.

## Introduction

Azure VM Boot Diagnostics captures the VM's serial console output and periodic screenshots during startup, enabling you to troubleshoot boot failures without console access. It also enables Azure Serial Console, which provides emergency interactive access to VMs even when SSH/RDP and the network are unavailable. Boot Diagnostics is free when using a managed storage account (enabled automatically) or can write to a custom storage account for longer retention.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials configured
- An Azure Resource Group and Storage Account (optional for custom storage)

## Step 1: Enable Boot Diagnostics with Managed Storage

```hcl
resource "azurerm_linux_virtual_machine" "main" {
  name                = "${var.project_name}-vm"
  resource_group_name = var.resource_group_name
  location            = var.location
  size                = "Standard_D2s_v3"

  network_interface_ids           = [azurerm_network_interface.main.id]
  admin_username                  = "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  # Enable boot diagnostics with Azure-managed storage (recommended)
  boot_diagnostics {
    # Omit storage_account_uri to use managed storage (free)
  }

  tags = {
    Name = "${var.project_name}-vm"
  }
}
```

## Step 2: Boot Diagnostics with Custom Storage Account

```hcl
resource "azurerm_storage_account" "boot_diagnostics" {
  name                     = "${var.project_name}bootdiag"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"  # LRS is sufficient for diagnostics

  # Allow public blob access for diagnostics portal viewing
  allow_nested_items_to_be_public = false

  # Lifecycle management for cost control
  blob_properties {
    delete_retention_policy {
      days = 30  # Keep boot diagnostic data for 30 days
    }
  }

  tags = {
    Name    = "${var.project_name}-boot-diag-storage"
    Purpose = "vm-boot-diagnostics"
  }
}

resource "azurerm_linux_virtual_machine" "with_custom_storage" {
  name                = "${var.project_name}-vm-diag"
  resource_group_name = var.resource_group_name
  location            = var.location
  size                = "Standard_D2s_v3"

  network_interface_ids           = [azurerm_network_interface.main.id]
  admin_username                  = "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.boot_diagnostics.primary_blob_endpoint
  }
}
```

## Step 3: Enable Boot Diagnostics on VM Scale Set

```hcl
resource "azurerm_linux_virtual_machine_scale_set" "main" {
  name                = "${var.project_name}-vmss"
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Standard_D2s_v3"
  instances           = 3

  admin_username                  = "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  network_interface {
    name    = "primary"
    primary = true

    ip_configuration {
      name      = "internal"
      primary   = true
      subnet_id = var.subnet_id
    }
  }

  # Boot diagnostics on scale set
  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.boot_diagnostics.primary_blob_endpoint
  }
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply

# View boot diagnostics (serial console output)

az vm boot-diagnostics get-boot-log \
  --resource-group <rg> \
  --name <vm-name>

# Get screenshot URL
az vm boot-diagnostics get-boot-log-uris \
  --resource-group <rg> \
  --name <vm-name>

# Access Serial Console in Azure Portal:
# VM > Help > Serial Console
# Or via Azure CLI:
az serial-console connect \
  --resource-group <rg> \
  --vm-name <vm-name>
```

## Conclusion

Always enable Boot Diagnostics in production-omit `storage_account_uri` to use Azure-managed storage at no cost without managing storage accounts. Boot Diagnostics is the prerequisite for Azure Serial Console, which provides emergency VM access when the OS hangs or SSH becomes unavailable. For Scale Sets, use a shared storage account rather than individual accounts per instance. Combine Boot Diagnostics with Azure Monitor Agent for comprehensive VM observability: Boot Diagnostics for startup/crash debugging, Azure Monitor for runtime metrics and logs.
