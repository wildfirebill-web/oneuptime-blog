# How to Set Up Azure Defender for Cloud with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Security, Defender, OpenTofu, Cloud Security, Compliance

Description: Learn how to enable and configure Azure Defender for Cloud (Microsoft Defender for Cloud) with OpenTofu to protect Azure resources with threat detection and security posture management.

## Overview

Microsoft Defender for Cloud provides unified security management and threat protection for Azure workloads. OpenTofu can enable Defender plans for specific resource types, configure security contacts, and set up auto-provisioning for the monitoring agent.

## Step 1: Enable Defender Plans

```hcl
# main.tf - Enable Microsoft Defender for specific resource types
# Defender for Servers
resource "azurerm_security_center_subscription_pricing" "defender_servers" {
  tier          = "Standard"  # "Free" or "Standard" (Defender enabled)
  resource_type = "VirtualMachines"
}

# Defender for SQL Servers on Machines
resource "azurerm_security_center_subscription_pricing" "defender_sql" {
  tier          = "Standard"
  resource_type = "SqlServers"
}

# Defender for Storage
resource "azurerm_security_center_subscription_pricing" "defender_storage" {
  tier          = "Standard"
  resource_type = "StorageAccounts"
}

# Defender for Kubernetes
resource "azurerm_security_center_subscription_pricing" "defender_k8s" {
  tier          = "Standard"
  resource_type = "KubernetesService"
}

# Defender for Container Registries
resource "azurerm_security_center_subscription_pricing" "defender_acr" {
  tier          = "Standard"
  resource_type = "ContainerRegistry"
}

# Defender for Key Vault
resource "azurerm_security_center_subscription_pricing" "defender_kv" {
  tier          = "Standard"
  resource_type = "KeyVaults"
}
```

## Step 2: Configure Security Contact

```hcl
# Security contact receives alerts and notifications
resource "azurerm_security_center_contact" "security_contact" {
  name  = "security-contact"
  email = "security@example.com"
  phone = "+1-555-555-5555"

  # Notify on high severity alerts
  alert_notifications = true

  # Send alerts to subscription owners
  alerts_to_admins = true
}
```

## Step 3: Configure Auto-Provisioning

```hcl
# Auto-provision the Log Analytics agent on new VMs
resource "azurerm_security_center_auto_provisioning" "mma" {
  auto_provision = "On"
}
```

## Step 4: Set Security Policy Workspace

```hcl
# Connect Defender data to a Log Analytics workspace
resource "azurerm_security_center_workspace" "law_workspace" {
  scope        = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  workspace_id = azurerm_log_analytics_workspace.law.id
}
```

## Step 5: Enable Defender for DevOps (GitHub/Azure DevOps)

```hcl
# Defender for DevOps security connector
resource "azurerm_security_center_setting" "mcas_integration" {
  setting_name = "MCAS"
  enabled      = true
}

resource "azurerm_security_center_setting" "wdatp_integration" {
  setting_name = "WDATP"
  enabled      = true
}
```

## Step 6: Outputs

```hcl
output "defender_enabled_plans" {
  value = [
    "VirtualMachines",
    "SqlServers",
    "StorageAccounts",
    "KubernetesService",
    "ContainerRegistry",
    "KeyVaults",
  ]
  description = "Microsoft Defender plans enabled on this subscription"
}
```

## Summary

Microsoft Defender for Cloud enabled via OpenTofu provides comprehensive threat protection across Azure services. By enabling Defender plans for specific resource types and configuring security contacts with auto-provisioning, you establish a security baseline that monitors for threats and provides remediation recommendations.
