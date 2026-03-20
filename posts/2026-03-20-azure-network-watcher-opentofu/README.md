# How to Configure Azure Network Watcher with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Network Watcher, Flow Logs, Diagnostics, Monitoring, Infrastructure as Code

Description: Learn how to configure Azure Network Watcher with OpenTofu to enable NSG flow logs, connection monitoring, and network diagnostic capabilities for your Azure infrastructure.

## Introduction

Azure Network Watcher provides tools to monitor, diagnose, and gain insights into Azure network infrastructure. Key features include NSG Flow Logs (capture all network flows through NSGs for traffic analysis), Connection Monitor (continuously test connectivity between sources and destinations), IP Flow Verify (test if traffic is allowed or denied by NSG rules), and Packet Capture (capture packets from VMs for deep analysis). Network Watcher is automatically enabled per region and requires no explicit creation.

## Prerequisites

- OpenTofu v1.6+
- Azure credentials configured
- NSGs and VMs to monitor

## Step 1: Enable NSG Flow Logs

```hcl
# Storage account for flow log data

resource "azurerm_storage_account" "flow_logs" {
  name                     = "${var.project_name}flowlogs"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  min_tls_version          = "TLS1_2"

  tags = {
    Name    = "${var.project_name}-flow-logs-storage"
    Purpose = "network-flow-logs"
  }
}

# Network Watcher (auto-created per region, import or reference)
data "azurerm_network_watcher" "main" {
  name                = "NetworkWatcher_${var.location}"
  resource_group_name = "NetworkWatcherRG"  # Default resource group
}

resource "azurerm_network_watcher_flow_log" "nsg" {
  network_watcher_name = data.azurerm_network_watcher.main.name
  resource_group_name  = data.azurerm_network_watcher.main.resource_group_name
  name                 = "${var.project_name}-nsg-flow-log"

  network_security_group_id = var.nsg_id
  storage_account_id        = azurerm_storage_account.flow_logs.id
  enabled                   = true
  version                   = 2  # Version 2 includes byte/packet counts

  retention_policy {
    enabled = true
    days    = 30  # Keep flow logs for 30 days
  }

  # Send to Log Analytics for Traffic Analytics
  traffic_analytics {
    enabled               = true
    workspace_id          = var.log_analytics_workspace_id
    workspace_region      = var.location
    workspace_resource_id = var.log_analytics_workspace_resource_id
    interval_in_minutes   = 10  # 10 or 60 minute aggregation intervals
  }

  tags = {
    Name = "${var.project_name}-flow-log"
  }
}
```

## Step 2: Connection Monitor

```hcl
resource "azurerm_network_connection_monitor" "main" {
  name               = "${var.project_name}-connection-monitor"
  network_watcher_id = data.azurerm_network_watcher.main.id
  location           = var.location

  endpoint {
    name               = "source-vm"
    target_resource_id = var.source_vm_id

    filter {
      item {
        address = var.source_vm_id
        type    = "AgentAddress"
      }
      type = "Include"
    }
  }

  endpoint {
    name    = "destination-sql"
    address = var.sql_server_fqdn
  }

  endpoint {
    name    = "destination-storage"
    address = "${var.storage_account_name}.blob.core.windows.net"
  }

  test_configuration {
    name                      = "tcp-test-sql"
    protocol                  = "Tcp"
    test_frequency_in_seconds = 30

    tcp_configuration {
      port = 1433
    }

    success_threshold {
      checks_failed_percent = 5
      round_trip_time_ms    = 100
    }
  }

  test_configuration {
    name                      = "https-test-storage"
    protocol                  = "Http"
    test_frequency_in_seconds = 60

    http_configuration {
      method = "Get"
      port   = 443
    }

    success_threshold {
      checks_failed_percent = 5
      round_trip_time_ms    = 500
    }
  }

  test_group {
    name                     = "sql-connectivity"
    destination_endpoints    = ["destination-sql"]
    source_endpoints         = ["source-vm"]
    test_configuration_names = ["tcp-test-sql"]
    enabled                  = true
  }

  test_group {
    name                     = "storage-connectivity"
    destination_endpoints    = ["destination-storage"]
    source_endpoints         = ["source-vm"]
    test_configuration_names = ["https-test-storage"]
    enabled                  = true
  }

  output_workspace_resource_ids = [var.log_analytics_workspace_resource_id]

  tags = {
    Name = "${var.project_name}-connection-monitor"
  }
}
```

## Step 3: Flow Log for All NSGs

```hcl
variable "nsg_ids" {
  description = "Map of NSG name to NSG ID"
  type        = map(string)
}

resource "azurerm_network_watcher_flow_log" "all_nsgs" {
  for_each = var.nsg_ids

  network_watcher_name      = data.azurerm_network_watcher.main.name
  resource_group_name       = data.azurerm_network_watcher.main.resource_group_name
  name                      = "${each.key}-flow-log"
  network_security_group_id = each.value
  storage_account_id        = azurerm_storage_account.flow_logs.id
  enabled                   = true
  version                   = 2

  retention_policy {
    enabled = true
    days    = 30
  }

  traffic_analytics {
    enabled               = true
    workspace_id          = var.log_analytics_workspace_id
    workspace_region      = var.location
    workspace_resource_id = var.log_analytics_workspace_resource_id
    interval_in_minutes   = 10
  }
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply

# IP flow verify - test if traffic is allowed
az network watcher test-ip-flow \
  --vm <vm-id> \
  --direction Inbound \
  --protocol TCP \
  --local 10.0.1.5:0 \
  --remote 203.0.113.1:443

# Next hop - trace routing path
az network watcher show-next-hop \
  --vm <vm-id> \
  --source-ip 10.0.1.5 \
  --dest-ip 8.8.8.8

# Initiate packet capture
az network watcher packet-capture create \
  --resource-group <rg> \
  --vm <vm-name> \
  --name capture1 \
  --storage-account <sa-name>
```

## Conclusion

Enable NSG Flow Logs version 2 with Traffic Analytics on all production NSGs-it provides insights into top talkers, allowed/denied flows, and unusual traffic patterns without requiring packet capture. Traffic Analytics requires a Log Analytics workspace and aggregates flow data every 10 or 60 minutes; use 10-minute intervals for near real-time visibility. Connection Monitor with the Network Watcher agent (installed via VM Extension) provides end-to-end latency and packet loss measurements between any two Azure or on-premises endpoints, making it the preferred tool for proactive connectivity monitoring.
