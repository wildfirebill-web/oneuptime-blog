# How to Deploy Dapr with Azure Bicep

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure Bicep, AKS, Azure, Infrastructure as Code

Description: Deploy Dapr on Azure Kubernetes Service using Azure Bicep for declarative, Azure-native infrastructure-as-code with ARM template generation.

---

## Azure Bicep for Dapr on AKS

Azure Bicep is a domain-specific language for deploying Azure resources declaratively. It compiles to ARM templates while offering a cleaner, more concise syntax. Combining Bicep with the AKS Dapr extension gives you a fully Azure-native way to install and manage Dapr.

## Prerequisites

```bash
# Install Azure CLI and Bicep
az upgrade
az bicep install

# Register required providers
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.KubernetesConfiguration
```

## AKS Cluster with Dapr Extension

Create `dapr-aks.bicep`:

```bicep
param location string = resourceGroup().location
param clusterName string = 'dapr-aks-cluster'
param nodeCount int = 3
param nodeVmSize string = 'Standard_D4s_v3'

// AKS Cluster
resource aksCluster 'Microsoft.ContainerService/managedClusters@2023-10-01' = {
  name: clusterName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    dnsPrefix: clusterName
    agentPoolProfiles: [
      {
        name: 'agentpool'
        count: nodeCount
        vmSize: nodeVmSize
        mode: 'System'
        osType: 'Linux'
      }
    ]
    kubernetesVersion: '1.29.0'
    enableRBAC: true
    networkProfile: {
      networkPlugin: 'azure'
      loadBalancerSku: 'Standard'
    }
  }
}

// Dapr Extension
resource daprExtension 'Microsoft.KubernetesConfiguration/extensions@2023-05-01' = {
  name: 'dapr'
  scope: aksCluster
  properties: {
    extensionType: 'microsoft.dapr'
    autoUpgradeMinorVersion: false
    releaseTrain: 'stable'
    version: '1.13.0'
    configurationSettings: {
      'global.mtls.enabled': 'true'
      'dapr_operator.replicaCount': '2'
      'dapr_sentry.replicaCount': '2'
      'dapr_placement.replicaCount': '3'
      'global.logAsJson': 'true'
    }
  }
}

output clusterName string = aksCluster.name
output clusterFqdn string = aksCluster.properties.fqdn
```

## Parameters File

Create `dapr-aks.parameters.json` for environment-specific values:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "clusterName": {
      "value": "dapr-prod-cluster"
    },
    "nodeCount": {
      "value": 5
    },
    "nodeVmSize": {
      "value": "Standard_D8s_v3"
    }
  }
}
```

## Deploying with Azure CLI

```bash
# Create resource group
az group create --name dapr-rg --location eastus

# Validate the Bicep template
az deployment group validate \
  --resource-group dapr-rg \
  --template-file dapr-aks.bicep \
  --parameters dapr-aks.parameters.json

# Deploy
az deployment group create \
  --resource-group dapr-rg \
  --template-file dapr-aks.bicep \
  --parameters dapr-aks.parameters.json

# Get credentials
az aks get-credentials --resource-group dapr-rg --name dapr-prod-cluster

# Verify Dapr
kubectl get pods -n dapr-system
```

## Adding Azure Key Vault Secret Store

Extend the Bicep template to add a Dapr secret store backed by Azure Key Vault:

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'dapr-keyvault-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
  }
}
```

## Summary

Azure Bicep combined with the AKS Dapr extension provides a fully Azure-native deployment path that requires no separate Helm operations. The extension model integrates with AKS lifecycle management, ensuring Dapr is automatically upgraded and reconciled as part of your cluster's managed lifecycle.
