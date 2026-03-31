# How to Use Azure Network Watcher for IPv6 Diagnostics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, IPv6, Network Watcher, Diagnostic, Flow Logs, Troubleshooting

Description: Use Azure Network Watcher tools to diagnose IPv6 connectivity issues, analyze IPv6 flow logs, verify NSG rules for IPv6 traffic, and trace IPv6 packet paths.

## Introduction

Azure Network Watcher provides a suite of diagnostic tools for IPv6 troubleshooting: IP flow verify checks if traffic is allowed or denied, connection troubleshoot tests end-to-end connectivity, and flow logs capture IPv6 traffic data. These tools are essential for diagnosing IPv6 connectivity issues without needing to access individual VMs.

## Enable Network Watcher

```bash
# Network Watcher is automatically created per region

# Verify it exists
az network watcher list \
    --query "[*].{name:name, region:location, status:provisioningState}"

# Create Network Watcher if not present
az network watcher configure \
    --resource-group NetworkWatcherRG \
    --locations eastus \
    --enabled true
```

## IP Flow Verify for IPv6 Traffic

```bash
# Test if IPv6 traffic is allowed from source to destination
VM_ID=$(az vm show \
    --resource-group "$RG" \
    --name vm-web-01 \
    --query id --output tsv)

NIC_ID=$(az vm show \
    --resource-group "$RG" \
    --name vm-web-01 \
    --query "networkProfile.networkInterfaces[0].id" \
    --output tsv)

# Verify inbound IPv6 HTTP traffic
az network watcher test-ip-flow \
    --direction Inbound \
    --protocol TCP \
    --local-ip "fd00:db8:0:1::10" \
    --local-port 80 \
    --remote-ip "2001:db8:client::1" \
    --remote-port 50000 \
    --nic "$NIC_ID" \
    --vm "$VM_ID"

# Output shows:
# access: Allow (or Deny)
# ruleName: The NSG rule that made the decision
```

## Connection Troubleshoot for IPv6

```bash
# Test connectivity between two VMs over IPv6
SOURCE_VM_ID=$(az vm show \
    --resource-group "$RG" \
    --name vm-source \
    --query id --output tsv)

DEST_VM_ID=$(az vm show \
    --resource-group "$RG" \
    --name vm-dest \
    --query id --output tsv)

az network watcher test-connectivity \
    --resource-group NetworkWatcherRG \
    --source-resource "$SOURCE_VM_ID" \
    --dest-resource "$DEST_VM_ID" \
    --dest-port 80 \
    --protocol TCP

# The output shows:
# connectionStatus: Reachable/Unreachable
# avgLatencyInMs: Round-trip latency
# hops: Each hop in the path with IPv6 addresses
```

## Enable IPv6 Flow Logs

```bash
# Create storage account for flow logs
az storage account create \
    --resource-group "$RG" \
    --name mystorageflowlogs \
    --sku Standard_LRS \
    --kind StorageV2

STORAGE_ID=$(az storage account show \
    --resource-group "$RG" \
    --name mystorageflowlogs \
    --query id --output tsv)

NSG_ID=$(az network nsg show \
    --resource-group "$RG" \
    --name nsg-web \
    --query id --output tsv)

# Enable flow logs (captures IPv4 and IPv6)
az network watcher flow-log create \
    --resource-group NetworkWatcherRG \
    --name flowlog-web \
    --nsg "$NSG_ID" \
    --storage-account "$STORAGE_ID" \
    --enabled true \
    --retention 7 \
    --traffic-analytics-workspace-id "/subscriptions/xxx/resourceGroups/rg-law/providers/Microsoft.OperationalInsights/workspaces/law-main" \
    --traffic-analytics-interval 10
```

## Analyze IPv6 Flow Logs

```bash
# Flow log JSON format (version 2)
# IPv6 entries have "::" in source/destination addresses

# Example flow log entry for IPv6:
# "flows": [{
#   "mac": "000D3A....",
#   "flowTuples": [
#     "1234567890,2001:db8::100,fd00:db8::10,50000,80,T,I,A,B,,,,"
#     # timestamp, src_ip, dst_ip, src_port, dst_port, protocol, direction, action
#   ]
# }]
```

## Terraform Flow Logs with Traffic Analytics

```hcl
# network_watcher.tf

data "azurerm_network_watcher" "main" {
  name                = "NetworkWatcher_eastus"
  resource_group_name = "NetworkWatcherRG"
}

resource "azurerm_network_watcher_flow_log" "web" {
  network_watcher_name = data.azurerm_network_watcher.main.name
  resource_group_name  = data.azurerm_network_watcher.main.resource_group_name
  name                 = "flowlog-web"

  network_security_group_id = azurerm_network_security_group.web.id
  storage_account_id        = azurerm_storage_account.flow_logs.id
  enabled                   = true

  retention_policy {
    enabled = true
    days    = 7
  }

  traffic_analytics {
    enabled               = true
    workspace_id          = azurerm_log_analytics_workspace.main.workspace_id
    workspace_region      = azurerm_log_analytics_workspace.main.location
    workspace_resource_id = azurerm_log_analytics_workspace.main.id
    interval_in_minutes   = 10
  }
}
```

## Query IPv6 Traffic in Log Analytics

```kusto
// KQL query for IPv6 flow logs in Log Analytics
AzureNetworkAnalytics_CL
| where TimeGenerated > ago(1h)
| where SrcIP_s contains ":" or DestIP_s contains ":"  // IPv6 addresses
| summarize count() by SrcIP_s, DestIP_s, DestPort_d, FlowDirection_s, FlowStatus_s
| order by count_ desc
| take 20
```

## Conclusion

Azure Network Watcher's IP flow verify tool (`az network watcher test-ip-flow`) tests whether specific IPv6 flows are allowed by NSGs, returning the exact rule name that permits or denies the traffic. Connection troubleshoot tests end-to-end IPv6 connectivity between VMs. NSG flow logs capture IPv6 traffic alongside IPv4, identifiable by the `:` in IP addresses. Use Log Analytics KQL queries with `SrcIP_s contains ":"` to filter IPv6 flows for security and performance analysis.
