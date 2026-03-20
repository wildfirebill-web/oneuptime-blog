# How to Set Up Azure Sentinel with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Sentinel, SIEM, OpenTofu, Security, Threat Detection

Description: Learn how to deploy and configure Microsoft Sentinel (Azure Sentinel) with OpenTofu for cloud-native SIEM and SOAR capabilities including data connectors and analytics rules.

## Overview

Microsoft Sentinel is a cloud-native SIEM (Security Information and Event Management) and SOAR (Security Orchestration, Automation and Response) solution. OpenTofu provisions the Sentinel workspace and configures data connectors, analytics rules, and watchlists.

## Step 1: Create Log Analytics Workspace

```hcl
# main.tf - Sentinel requires a Log Analytics workspace

resource "azurerm_log_analytics_workspace" "sentinel_law" {
  name                = "sentinel-workspace"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 90
}

# Enable Microsoft Sentinel on the workspace
resource "azurerm_sentinel_log_analytics_workspace_onboarding" "sentinel" {
  workspace_id = azurerm_log_analytics_workspace.sentinel_law.id
}
```

## Step 2: Enable Data Connectors

```hcl
# Enable Azure Active Directory connector
resource "azurerm_sentinel_data_connector_azure_active_directory" "aad_connector" {
  name                       = "aad-connector"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.sentinel_law.id
  tenant_id                  = data.azurerm_client_config.current.tenant_id
}

# Enable Azure Security Center connector
resource "azurerm_sentinel_data_connector_azure_security_center" "asc_connector" {
  name                       = "asc-connector"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.sentinel_law.id
  subscription_id            = data.azurerm_client_config.current.subscription_id
}

# Enable Microsoft Defender for Cloud connector
resource "azurerm_sentinel_data_connector_microsoft_defender_advanced_threat_protection" "mde_connector" {
  name                       = "mde-connector"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.sentinel_law.id
  tenant_id                  = data.azurerm_client_config.current.tenant_id
}

# Enable Office 365 connector
resource "azurerm_sentinel_data_connector_office_365" "o365_connector" {
  name                       = "o365-connector"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.sentinel_law.id
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  exchange_enabled           = true
  sharepoint_enabled         = true
  teams_enabled              = true
}
```

## Step 3: Create Analytics Rules

```hcl
# Scheduled analytics rule to detect suspicious sign-ins
resource "azurerm_sentinel_alert_rule_scheduled" "suspicious_signin" {
  name                       = "suspicious-signin-rule"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.sentinel_law.id
  display_name               = "Suspicious Sign-in from Multiple Countries"
  severity                   = "High"
  enabled                    = true

  # KQL query to detect the threat
  query = <<-KQL
    SigninLogs
    | where TimeGenerated > ago(1h)
    | summarize Countries = make_set(LocationDetails.countryOrRegion),
                CountryCount = dcount(LocationDetails.countryOrRegion) by UserPrincipalName
    | where CountryCount >= 3
    | project UserPrincipalName, Countries, CountryCount
  KQL

  query_frequency = "PT1H"   # Run every hour
  query_period    = "PT1H"   # Look back 1 hour

  trigger_operator  = "GreaterThan"
  trigger_threshold = 0

  tactics = ["InitialAccess", "CredentialAccess"]
}
```

## Step 4: Create a Watchlist

```hcl
# Watchlist of known malicious IP addresses
resource "azurerm_sentinel_watchlist" "malicious_ips" {
  name                       = "MaliciousIPs"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.sentinel_law.id
  display_name               = "Known Malicious IP Addresses"
  item_search_key            = "IPAddress"
  description                = "Watchlist of known malicious IP addresses for threat detection"
}
```

## Step 5: Outputs

```hcl
output "sentinel_workspace_id" {
  value       = azurerm_log_analytics_workspace.sentinel_law.workspace_id
  description = "Log Analytics workspace ID for Sentinel"
}

output "sentinel_workspace_name" {
  value = azurerm_log_analytics_workspace.sentinel_law.name
}
```

## Summary

Microsoft Sentinel deployed with OpenTofu provides a fully operational SIEM from day one. By enabling data connectors for Azure AD, Security Center, and Office 365, and defining analytics rules as KQL queries, you create a continuous threat detection pipeline that surfaces security incidents automatically.
