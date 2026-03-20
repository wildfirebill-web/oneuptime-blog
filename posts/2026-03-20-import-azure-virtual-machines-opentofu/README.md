# How to Import Azure Virtual Machines into OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Azure, Virtual Machines, Import, Compute

Description: Learn how to import existing Azure Virtual Machines into OpenTofu state, including network interfaces, managed disks, and OS disk configurations.

## Introduction

Azure VMs are complex resources that consist of the VM itself plus associated network interfaces, managed disks, and availability set configurations. This guide covers importing the full VM stack.

## Step 1: Gather VM Configuration

```bash
RG="my-app-rg"
VM_NAME="my-app-vm"

# Get VM details
az vm show --resource-group $RG --name $VM_NAME --output json | jq '{
  location: .location,
  vm_size: .hardwareProfile.vmSize,
  os_type: .storageProfile.osDisk.osType,
  image: .storageProfile.imageReference,
  nic_ids: [.networkProfile.networkInterfaces[].id]
}'

# Get the NIC details
NIC_NAME=$(az vm show -g $RG -n $VM_NAME --query 'networkProfile.networkInterfaces[0].id' -o tsv | xargs basename)
az network nic show --resource-group $RG --name $NIC_NAME --output json | jq '{
  subnet_id: .ipConfigurations[0].subnet.id,
  private_ip: .ipConfigurations[0].privateIPAddress
}'
```

## Step 2: Write Matching HCL

```hcl
# Network interface must be imported first
resource "azurerm_network_interface" "app" {
  name                = "my-app-vm-nic"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location

  ip_configuration {
    name                          = "ipconfig1"
    subnet_id                     = var.subnet_id
    private_ip_address_allocation = "Dynamic"
  }

  tags = { Environment = "prod" }
}

resource "azurerm_linux_virtual_machine" "app" {
  name                = "my-app-vm"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  size                = "Standard_D2s_v3"
  admin_username      = "azureuser"

  network_interface_ids = [azurerm_network_interface.app.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 30
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  tags = { Environment = "prod", ManagedBy = "OpenTofu" }

  lifecycle {
    # Prevent replacement when source image version changes
    ignore_changes = [source_image_reference]
  }
}
```

## Import Blocks

```hcl
# import.tf
import {
  to = azurerm_network_interface.app
  id = "/subscriptions/SUBSCRIPTION_ID/resourceGroups/my-app-rg/providers/Microsoft.Network/networkInterfaces/my-app-vm-nic"
}

import {
  to = azurerm_linux_virtual_machine.app
  id = "/subscriptions/SUBSCRIPTION_ID/resourceGroups/my-app-rg/providers/Microsoft.Compute/virtualMachines/my-app-vm"
}
```

## Importing Managed Data Disks

```hcl
resource "azurerm_managed_disk" "data" {
  name                 = "my-app-vm-data-disk"
  resource_group_name  = azurerm_resource_group.app.name
  location             = azurerm_resource_group.app.location
  storage_account_type = "Premium_LRS"
  create_option        = "Empty"
  disk_size_gb         = 100
}

resource "azurerm_virtual_machine_data_disk_attachment" "data" {
  managed_disk_id    = azurerm_managed_disk.data.id
  virtual_machine_id = azurerm_linux_virtual_machine.app.id
  lun                = 0
  caching            = "ReadWrite"
}

import {
  to = azurerm_managed_disk.data
  id = "/subscriptions/SUBSCRIPTION_ID/resourceGroups/my-app-rg/providers/Microsoft.Compute/disks/my-app-vm-data-disk"
}
```

## Conclusion

Azure VM import requires importing the NIC before the VM since the VM references the NIC ID. Use `ignore_changes = [source_image_reference]` to prevent OpenTofu from planning a VM replacement every time a new image version is published. For Windows VMs, use `azurerm_windows_virtual_machine` with the `admin_password` attribute.
