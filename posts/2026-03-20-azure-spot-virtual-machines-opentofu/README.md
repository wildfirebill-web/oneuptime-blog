# How to Configure Azure Spot Virtual Machines with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Spot VMs, Cost Optimization, Interruptible Workloads, Infrastructure as Code

Description: Learn how to configure Azure Spot Virtual Machines with OpenTofu to run fault-tolerant workloads at up to 90% cost savings compared to regular on-demand VMs.

## Introduction

Azure Spot Virtual Machines use Azure's excess compute capacity at up to 90% discount compared to regular VMs. In exchange, Azure can evict Spot VMs with only a 30-second warning when it needs the capacity back. Spot VMs are ideal for batch processing, data analytics, CI/CD pipelines, stateless web frontends, and any workload that can tolerate interruption and restart. They are not suitable for databases, production stateful services, or SLA-critical applications.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials configured
- A Resource Group and Virtual Network
- Workloads designed for interruption handling

## Step 1: Create a Spot VM

```hcl
resource "azurerm_linux_virtual_machine" "spot" {
  name                = "${var.project_name}-spot-vm"
  resource_group_name = var.resource_group_name
  location            = var.location
  size                = "Standard_D4s_v3"

  # Enable Spot pricing
  priority        = "Spot"
  eviction_policy = "Deallocate"  # Deallocate or Delete

  # Optional: set max price (in USD/hour)
  # Set to -1 to pay up to on-demand price (default)
  max_bid_price = -1

  network_interface_ids           = [azurerm_network_interface.spot.id]
  admin_username                  = "azureuser"
  disable_password_authentication = true

  admin_ssh_key {
    username   = "azureuser"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"  # Use Standard for Spot cost savings
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  tags = {
    Name     = "${var.project_name}-spot-vm"
    Priority = "Spot"
  }
}
```

## Step 2: Spot VM Scale Set for Batch Workloads

```hcl
resource "azurerm_linux_virtual_machine_scale_set" "spot" {
  name                = "${var.project_name}-spot-vmss"
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Standard_D4s_v3"
  instances           = 5

  # Spot configuration
  priority        = "Spot"
  eviction_policy = "Delete"   # Delete VMs when evicted (cheaper - no disk charges)
  max_bid_price   = 0.04       # Max price in USD/hour per instance

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
    storage_account_type = "Standard_LRS"
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

  # Spread across zones for better capacity availability
  zones = ["1", "2", "3"]

  # Rebalance Spot capacity across zones
  spot_restore {
    enabled = true
    timeout = "PT1H"  # Try to restore for 1 hour after eviction
  }
}
```

## Step 3: Handle Eviction Events

```hcl
# Azure Scheduled Events listener (run on the VM)
# Monitor: http://169.254.169.254/metadata/scheduledevents

resource "azurerm_virtual_machine_extension" "eviction_handler" {
  name                 = "eviction-handler"
  virtual_machine_id   = azurerm_linux_virtual_machine.spot.id
  publisher            = "Microsoft.Azure.Extensions"
  type                 = "CustomScript"
  type_handler_version = "2.1"

  settings = jsonencode({
    script = base64encode(<<-EOT
      #!/bin/bash
      # Install eviction handler daemon
      cat > /usr/local/bin/spot-eviction-handler.sh <<'HANDLER'
      #!/bin/bash
      while true; do
        EVENTS=$(curl -s -H "Metadata: true" \
          "http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01")
        if echo "$EVENTS" | grep -q "Preempt"; then
          # Graceful shutdown: drain tasks, save state, flush logs
          systemctl stop app-worker.service
          aws s3 sync /var/data/checkpoint s3://checkpoint-bucket/
          break
        fi
        sleep 5
      done
      HANDLER
      chmod +x /usr/local/bin/spot-eviction-handler.sh
      nohup /usr/local/bin/spot-eviction-handler.sh &
    EOT
    )
  })
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check current Spot VM state
az vm show \
  --resource-group <rg> \
  --name <vm-name> \
  --query "{Priority: priority, EvictionPolicy: evictionPolicy, State: provisioningState}"

# Check Spot pricing for a VM size
az vm list-skus \
  --location eastus \
  --size Standard_D4s_v3 \
  --query "[].{Name: name, SpotAvailable: capabilities[?name=='LowPriorityCapable'].value}"
```

## Conclusion

Use `eviction_policy = "Delete"` for stateless batch workloads (avoids ongoing disk charges after eviction) and `eviction_policy = "Deallocate"` for workloads that need to resume from where they stopped. Setting `max_bid_price = -1` means you pay up to the on-demand price and reduce eviction frequency—useful for longer batch jobs. For VM Scale Sets, deploy across multiple zones with `spot_restore` enabled to automatically replace evicted instances. Always poll the Azure Scheduled Events endpoint (http://169.254.169.254/metadata/scheduledevents) from within Spot VMs to get the 30-second eviction warning and save checkpoint state.
