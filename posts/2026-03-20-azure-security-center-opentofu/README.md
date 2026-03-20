# How to Configure Azure Security Center with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Security Center, OpenTofu, Security, Compliance, Governance

Description: Learn how to configure Azure Security Center (Microsoft Defender for Cloud) with OpenTofu to enable security posture management, compliance assessments, and alert notifications.

## Overview

Azure Security Center (now part of Microsoft Defender for Cloud) provides unified security management with continuous security assessment, actionable recommendations, and threat protection. OpenTofu manages the core configuration settings.

## Step 1: Configure Security Center Pricing

```hcl
# main.tf - Enable Defender for Cloud standard tier on key resource types

resource "azurerm_security_center_subscription_pricing" "defender_vms" {
  tier          = "Standard"
  resource_type = "VirtualMachines"
}

resource "azurerm_security_center_subscription_pricing" "defender_sql" {
  tier          = "Standard"
  resource_type = "SqlServers"
}

resource "azurerm_security_center_subscription_pricing" "defender_app_services" {
  tier          = "Standard"
  resource_type = "AppServices"
}
```

## Step 2: Security Contact Configuration

```hcl
# Configure who receives security alerts
resource "azurerm_security_center_contact" "contact" {
  name  = "primary-security-contact"
  email = "security-team@example.com"
  phone = "+1-555-0100"

  # Receive high severity alerts via email
  alert_notifications = true

  # Also notify subscription owners and co-admins
  alerts_to_admins = true
}
```

## Step 3: Auto-Provisioning of Security Agents

```hcl
# Automatically install the monitoring agent on new VMs
resource "azurerm_security_center_auto_provisioning" "auto_provision" {
  auto_provision = "On"  # "Off" to disable
}
```

## Step 4: Connect to Log Analytics Workspace

```hcl
# Log Analytics workspace to receive Security Center data
resource "azurerm_log_analytics_workspace" "security_law" {
  name                = "security-center-workspace"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 90
}

# Link Security Center to the workspace
resource "azurerm_security_center_workspace" "sc_workspace" {
  scope        = "/subscriptions/${data.azurerm_subscription.current.subscription_id}"
  workspace_id = azurerm_log_analytics_workspace.security_law.id
}
```

## Step 5: Configure Security Alerts Integration

```hcl
# Send Security Center alerts to a Logic App for notification workflow
resource "azurerm_monitor_action_group" "security_alerts" {
  name                = "security-center-alerts"
  resource_group_name = azurerm_resource_group.rg.name
  short_name          = "seculerts"

  email_receiver {
    name          = "security-team"
    email_address = "security-team@example.com"
  }

  webhook_receiver {
    name        = "pagerduty"
    service_uri = var.pagerduty_webhook_url
  }
}
```

## Step 6: Policy Assignment for CIS Compliance

```hcl
# Assign Azure Security Benchmark policy initiative
resource "azurerm_subscription_policy_assignment" "azure_security_benchmark" {
  name                 = "azure-security-benchmark"
  subscription_id      = data.azurerm_subscription.current.id
  # Built-in Azure Security Benchmark initiative
  policy_definition_id = "/providers/Microsoft.Authorization/policySetDefinitions/1f3afdf9-d0c9-4c3d-847f-89da613e70a8"
  display_name         = "Azure Security Benchmark"
}
```

## Summary

Azure Security Center configured with OpenTofu establishes a security baseline for your Azure subscription. Auto-provisioning ensures all VMs get the monitoring agent, the Log Analytics workspace centralizes security data, and security contacts ensure the right people are notified when threats are detected.
