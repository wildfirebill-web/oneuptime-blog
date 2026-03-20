# How to Implement CIS Benchmark Controls with OpenTofu on Azure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, CIS Benchmark, Azure Security, Compliance, Infrastructure as Code

Description: Learn how to implement CIS Microsoft Azure Foundations Benchmark controls with OpenTofu to establish a security baseline for Azure subscriptions.

The CIS Microsoft Azure Foundations Benchmark provides security guidance for Azure subscriptions. OpenTofu lets you codify these controls as Azure Policy assignments, security center settings, and resource configurations.

## Section 1: Identity and Access Management

```hcl
# CIS 1.1 - Enable Security Defaults or Conditional Access (requires Entra ID P1)

# CIS 1.22 - Ensure that 'No users are allowed to consent to apps accessing company data'
resource "azurerm_resource_group_policy_assignment" "no_app_consent" {
  name                 = "restrict-app-consent"
  resource_group_id    = azurerm_resource_group.security.id
  policy_definition_id = "/providers/Microsoft.Authorization/policyDefinitions/..."
  description          = "CIS 1.22 - Restrict user app consent"
}
```

## Section 2: Microsoft Defender for Cloud

```hcl
# CIS 2.1 - Enable Defender for Servers
resource "azurerm_security_center_subscription_pricing" "servers" {
  tier          = "Standard"
  resource_type = "VirtualMachines"
}

# CIS 2.2 - Enable Defender for App Services
resource "azurerm_security_center_subscription_pricing" "app_services" {
  tier          = "Standard"
  resource_type = "AppServices"
}

# CIS 2.3 - Enable Defender for SQL Servers
resource "azurerm_security_center_subscription_pricing" "sql" {
  tier          = "Standard"
  resource_type = "SqlServers"
}

# CIS 2.4 - Enable Defender for Storage
resource "azurerm_security_center_subscription_pricing" "storage" {
  tier          = "Standard"
  resource_type = "StorageAccounts"
}

# CIS 2.5 - Enable Defender for Kubernetes
resource "azurerm_security_center_subscription_pricing" "kubernetes" {
  tier          = "Standard"
  resource_type = "KubernetesService"
}
```

## Section 3: Storage Accounts

```hcl
# CIS 3.1 - Ensure that 'Secure transfer required' is set to 'Enabled'
resource "azurerm_storage_account" "cis_compliant" {
  name                     = "mycompliantstorage"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = "eastus"
  account_tier             = "Standard"
  account_replication_type = "GRS"

  enable_https_traffic_only       = true  # CIS 3.1
  min_tls_version                 = "TLS1_2"  # CIS 3.2
  allow_nested_items_to_be_public = false     # CIS 3.5
}

# CIS 3.7 - Ensure storage account uses private endpoints
resource "azurerm_private_endpoint" "storage" {
  name                = "storage-private-endpoint"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.private.id

  private_service_connection {
    name                           = "storage-connection"
    private_connection_resource_id = azurerm_storage_account.cis_compliant.id
    subresource_names              = ["blob"]
    is_manual_connection           = false
  }
}
```

## Section 4: Database Services

```hcl
# CIS 4.1.1 - Ensure SQL Server audit is enabled
resource "azurerm_mssql_server_extended_auditing_policy" "main" {
  server_id                               = azurerm_mssql_server.main.id
  storage_endpoint                        = azurerm_storage_account.audit.primary_blob_endpoint
  storage_account_access_key              = azurerm_storage_account.audit.primary_access_key
  storage_account_access_key_is_secondary = false
  retention_in_days                       = 90
}
```

## Section 5: Logging and Monitoring

```hcl
# CIS 5.1.1 - Ensure Diagnostic Setting captures all activities
resource "azurerm_monitor_diagnostic_setting" "subscription" {
  name               = "subscription-diag"
  target_resource_id = "/subscriptions/${var.subscription_id}"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.security.id

  enabled_log { category = "Administrative" }
  enabled_log { category = "Security" }
  enabled_log { category = "ServiceHealth" }
  enabled_log { category = "Alert" }
  enabled_log { category = "Recommendation" }
  enabled_log { category = "Policy" }
  enabled_log { category = "Autoscale" }
  enabled_log { category = "ResourceHealth" }
}
```

## Section 6: Networking

```hcl
# CIS 6.3 - Ensure SSH access is restricted to known IPs
resource "azurerm_network_security_rule" "deny_ssh_all" {
  name                        = "deny-ssh-internet"
  priority                    = 4096
  direction                   = "Inbound"
  access                      = "Deny"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "22"
  source_address_prefix       = "Internet"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.main.name
  network_security_group_name = azurerm_network_security_group.main.name
}
```

## Conclusion

CIS Azure Benchmark controls span IAM, Defender for Cloud, storage, databases, logging, and networking. Use `azurerm_security_center_subscription_pricing` to enable Defender plans, enforce storage security settings at resource creation, and use Azure Policy assignments for organization-wide enforcement. Enable diagnostic settings to feed activity logs to Log Analytics for SIEM integration.
