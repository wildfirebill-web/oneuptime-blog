# How to Configure Azure VM Scale Sets with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, VM Scale Sets, VMSS, Infrastructure as Code, Compute

Description: Learn how to configure Azure Virtual Machine Scale Sets with OpenTofu - including autoscaling profiles, rolling upgrades, load balancer integration, and custom script extensions.

## Introduction

Azure Virtual Machine Scale Sets (VMSS) manage a group of identical VMs that scale automatically. OpenTofu manages the scale set definition, autoscale settings, health extensions, and integration with Azure Load Balancer or Application Gateway.

## Linux VMSS

```hcl
resource "azurerm_linux_virtual_machine_scale_set" "app" {
  name                = "${var.environment}-app-vmss"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  sku                 = var.vm_sku
  instances           = var.instance_count
  admin_username      = "azureuser"

  admin_ssh_key {
    username   = "azureuser"
    public_key = tls_private_key.ssh.public_key_openssh
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts-gen2"
    version   = "latest"
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 30
  }

  network_interface {
    name    = "nic"
    primary = true

    ip_configuration {
      name                                   = "ipconfig"
      primary                                = true
      subnet_id                              = azurerm_subnet.app.id
      load_balancer_backend_address_pool_ids = [azurerm_lb_backend_address_pool.app.id]
    }
  }

  identity {
    type = "SystemAssigned"
  }

  upgrade_mode = "Rolling"

  rolling_upgrade_policy {
    max_batch_instance_percent              = 20
    max_unhealthy_instance_percent          = 20
    max_unhealthy_upgraded_instance_percent = 5
    pause_time_between_batches              = "PT0S"
  }

  health_probe_id = azurerm_lb_probe.app.id

  custom_data = base64encode(file("${path.module}/cloud-init.yaml"))

  tags = { Environment = var.environment }
}
```

## Autoscale Settings

```hcl
resource "azurerm_monitor_autoscale_setting" "app" {
  name                = "${var.environment}-app-autoscale"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  target_resource_id  = azurerm_linux_virtual_machine_scale_set.app.id

  profile {
    name = "default"

    capacity {
      default = 2
      minimum = 2
      maximum = 20
    }

    # Scale out when CPU > 75%
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.app.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 75
      }

      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }

    # Scale in when CPU < 25%
    rule {
      metric_trigger {
        metric_name        = "Percentage CPU"
        metric_resource_id = azurerm_linux_virtual_machine_scale_set.app.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT10M"
        time_aggregation   = "Average"
        operator           = "LessThan"
        threshold          = 25
      }

      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT10M"
      }
    }
  }

  # Scheduled profile for business hours
  profile {
    name = "business-hours"

    capacity {
      default = 4
      minimum = 4
      maximum = 20
    }

    recurrence {
      timezone = "UTC"
      days     = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
      hours    = [7]
      minutes  = [0]
    }
  }

  notification {
    email {
      send_to_subscription_administrator = true
    }
  }
}
```

## Load Balancer Integration

```hcl
resource "azurerm_lb" "app" {
  name                = "${var.environment}-app-lb"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  sku                 = "Standard"

  frontend_ip_configuration {
    name                 = "frontend"
    public_ip_address_id = azurerm_public_ip.lb.id
  }
}

resource "azurerm_lb_backend_address_pool" "app" {
  loadbalancer_id = azurerm_lb.app.id
  name            = "app-pool"
}

resource "azurerm_lb_probe" "app" {
  loadbalancer_id = azurerm_lb.app.id
  name            = "http-probe"
  protocol        = "Http"
  port            = 80
  request_path    = "/health"
  interval_in_seconds = 15
  number_of_probes    = 3
}

resource "azurerm_lb_rule" "app" {
  loadbalancer_id                = azurerm_lb.app.id
  name                           = "http"
  protocol                       = "Tcp"
  frontend_port                  = 80
  backend_port                   = 80
  frontend_ip_configuration_name = "frontend"
  backend_address_pool_ids       = [azurerm_lb_backend_address_pool.app.id]
  probe_id                       = azurerm_lb_probe.app.id
  disable_outbound_snat          = true
}
```

## Custom Script Extension

```hcl
resource "azurerm_virtual_machine_scale_set_extension" "app_setup" {
  name                         = "app-setup"
  virtual_machine_scale_set_id = azurerm_linux_virtual_machine_scale_set.app.id
  publisher                    = "Microsoft.Azure.Extensions"
  type                         = "CustomScript"
  type_handler_version         = "2.1"

  settings = jsonencode({
    commandToExecute = "bash setup.sh"
    fileUris = [
      "https://${azurerm_storage_account.scripts.name}.blob.core.windows.net/scripts/setup.sh"
    ]
  })

  protected_settings = jsonencode({
    storageAccountName = azurerm_storage_account.scripts.name
    storageAccountKey  = azurerm_storage_account.scripts.primary_access_key
  })
}
```

## Conclusion

Azure VMSS with OpenTofu provides elastic compute for Azure workloads. Use `upgrade_mode = "Rolling"` with `health_probe_id` so Azure safely updates instances without downtime. The `azurerm_monitor_autoscale_setting` resource supports both metric-based rules (CPU, memory, custom metrics) and scheduled profiles for predictable traffic patterns. Use `SystemAssigned` managed identity on the VMSS to grant access to Azure Key Vault, Storage, and other services without storing credentials.
