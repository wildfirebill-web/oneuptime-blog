# How to Create a Virtual Machine with OpenTofu on Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Virtual Machine, Compute, Infrastructure as Code

Description: Learn how to create an Azure Virtual Machine with OpenTofu including OS disk, network interface, and SSH key configuration.

## Introduction

Azure Virtual Machines provide IaaS compute for workloads that require full OS control. This guide creates a Linux VM with managed OS disk, public IP, and network interface.

## Core Configuration

```hcl
resource "azurerm_linux_virtual_machine" "main" {
  name                = "vm-${var.name}-${var.environment}"
  resource_group_name = var.resource_group_name
  location            = var.location
  size                = var.vm_size  # e.g., "Standard_B2s"
  admin_username      = "azureuser"

  network_interface_ids = [azurerm_network_interface.main.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 64
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  identity {
    type = "SystemAssigned"
  }

  tags = { Name = var.name, Environment = var.environment }
}
```

## Variables

```hcl
variable "resource_group_name" { type = string }
variable "location"            { type = string; default = "East US" }
variable "environment"         { type = string }
variable "name"                { type = string }
```

## Outputs

```hcl
output "id"   { value = azurerm_resource_type.main.id }
output "name" { value = azurerm_resource_type.main.name }
```

## Conclusion

Managing Azure resources with OpenTofu provides reproducible, version-controlled infrastructure. Always tag resources consistently and use separate resource groups per tier for cost allocation and access control.
