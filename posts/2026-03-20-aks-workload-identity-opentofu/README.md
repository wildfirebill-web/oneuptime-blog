# How to Configure AKS Workload Identity with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, AKS, Kubernetes, OpenTofu, Identity, Security

Description: Learn how to configure AKS Workload Identity with OpenTofu to enable pods to authenticate with Azure services without managing secrets.

## What is AKS Workload Identity?

AKS Workload Identity allows Kubernetes pods to authenticate with Azure services using Azure Active Directory (AAD) identities, replacing the need for application-level secrets or connection strings. It leverages OpenID Connect (OIDC) federation to bind a Kubernetes service account to an Azure managed identity.

## Prerequisites

- OpenTofu installed (>= 1.6)
- Azure CLI configured
- An existing AKS cluster or one you want to create

## Step 1: Enable OIDC Issuer and Workload Identity on AKS

First, create or update an AKS cluster with OIDC issuer and workload identity enabled.

```hcl
# main.tf - AKS cluster with OIDC and Workload Identity enabled
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "my-aks-cluster"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "myaks"

  # Enable OIDC issuer URL for workload identity federation
  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  default_node_pool {
    name       = "default"
    node_count = 2
    vm_size    = "Standard_D2s_v3"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

## Step 2: Create a User-Assigned Managed Identity

```hcl
# Create the managed identity that workloads will assume
resource "azurerm_user_assigned_identity" "workload_identity" {
  name                = "my-app-identity"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Grant the identity access to Azure Key Vault (example permission)
resource "azurerm_role_assignment" "kv_reader" {
  scope                = azurerm_key_vault.kv.id
  role_definition_name = "Key Vault Secrets User"
  principal_id         = azurerm_user_assigned_identity.workload_identity.principal_id
}
```

## Step 3: Configure Federated Identity Credentials

This step links the Kubernetes service account to the managed identity via OIDC federation.

```hcl
# Federated credential binds the K8s service account to the Azure managed identity
resource "azurerm_federated_identity_credential" "workload_fed" {
  name                = "my-app-federated-credential"
  resource_group_name = azurerm_resource_group.rg.name
  parent_id           = azurerm_user_assigned_identity.workload_identity.id

  # The OIDC issuer URL from the AKS cluster
  issuer = azurerm_kubernetes_cluster.aks.oidc_issuer_url

  # The Kubernetes service account subject
  subject = "system:serviceaccount:default:my-app-sa"

  # Azure AD audience for workload identity
  audience = ["api://AzureADTokenExchange"]
}
```

## Step 4: Deploy the Kubernetes Service Account

```hcl
# Deploy the annotated Kubernetes service account via OpenTofu
resource "kubernetes_service_account" "app_sa" {
  metadata {
    name      = "my-app-sa"
    namespace = "default"

    # These annotations link the service account to the managed identity
    annotations = {
      "azure.workload.identity/client-id" = azurerm_user_assigned_identity.workload_identity.client_id
    }

    labels = {
      "azure.workload.identity/use" = "true"
    }
  }
}
```

## Step 5: Output the OIDC Issuer URL

```hcl
# Output the OIDC issuer URL for reference
output "oidc_issuer_url" {
  value       = azurerm_kubernetes_cluster.aks.oidc_issuer_url
  description = "OIDC issuer URL for the AKS cluster"
}

output "managed_identity_client_id" {
  value       = azurerm_user_assigned_identity.workload_identity.client_id
  description = "Client ID to annotate on Kubernetes service accounts"
}
```

## How It Works

When a pod uses the annotated service account, the AKS workload identity webhook injects the OIDC token as an environment variable. The application SDK uses this token to exchange for an Azure AD access token via the federated credential, granting access to Azure services without any stored secrets.

## Summary

AKS Workload Identity with OpenTofu provides a secure, secret-free way to authenticate pods with Azure services. By combining OIDC federation, managed identities, and Kubernetes service account annotations, you eliminate credential management overhead while improving your security posture.
