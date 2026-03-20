# How to Configure Azure Backend with Managed Identity in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backend, Azure

Description: Learn how to configure the OpenTofu Azure backend with Managed Identity for keyless authentication when running on Azure-hosted compute.

## Introduction

Managed Identity is the preferred authentication method for the Azure backend when OpenTofu runs on Azure-hosted compute (Azure VMs, AKS, Azure DevOps agents, Container Instances). No credentials need to be stored or rotated - Azure provides the identity token automatically.

## System-Assigned Managed Identity

Enable a system-assigned identity on your compute resource:

```bash
# Enable system-assigned identity on a VM

az vm identity assign \
  --name my-terraform-vm \
  --resource-group my-rg

# Get the principal ID
az vm show \
  --name my-terraform-vm \
  --resource-group my-rg \
  --query identity.principalId \
  --output tsv
```

Grant the identity access to the state storage account:

```bash
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee "PRINCIPAL_ID" \
  --scope "/subscriptions/SUB_ID/resourceGroups/terraform-state-rg/providers/Microsoft.Storage/storageAccounts/acmecorptofu"
```

## Backend Configuration for Managed Identity

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "acmecorptofu"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"

    use_msi = true  # Use Managed Service Identity
    # subscription_id and tenant_id are auto-detected from the identity
  }
}
```

## User-Assigned Managed Identity

For more control, use a user-assigned identity that can be shared across resources:

```hcl
# Create user-assigned identity
resource "azurerm_user_assigned_identity" "tofu" {
  name                = "opentofu-state-identity"
  location            = azurerm_resource_group.tofu_state.location
  resource_group_name = azurerm_resource_group.tofu_state.name
}

# Grant access to state storage
resource "azurerm_role_assignment" "tofu_state" {
  scope                = azurerm_storage_account.tofu_state.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_user_assigned_identity.tofu.principal_id
}
```

```hcl
# Backend configuration with user-assigned identity
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "acmecorptofu"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"

    use_msi   = true
    client_id = "USER_ASSIGNED_IDENTITY_CLIENT_ID"
  }
}
```

## AKS with Workload Identity

For AKS deployments using Workload Identity:

```hcl
# Create federated identity credential for AKS
resource "azurerm_federated_identity_credential" "tofu" {
  name                = "tofu-aks-federated"
  resource_group_name = azurerm_resource_group.tofu.name
  parent_id           = azurerm_user_assigned_identity.tofu.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = azurerm_kubernetes_cluster.main.oidc_issuer_url
  subject             = "system:serviceaccount:terraform:tofu-runner"
}
```

```bash
# In the AKS pod, no credentials needed
# The Workload Identity webhook injects the token automatically
tofu init
tofu apply
```

## Azure DevOps Pipeline

```yaml
# azure-pipelines.yml - Managed Identity on self-hosted agent
pool:
  name: SelfHostedAgentsWithManagedIdentity

steps:
  - task: OpenTofuInstaller@0
    inputs:
      tofuVersion: '1.8.0'

  - script: |
      tofu init
      tofu plan
      tofu apply -auto-approve
    displayName: Deploy Infrastructure
    env:
      ARM_USE_MSI: "true"
      ARM_SUBSCRIPTION_ID: $(SUBSCRIPTION_ID)
      ARM_TENANT_ID: $(TENANT_ID)
```

## Conclusion

Managed Identity authentication eliminates credential management for Azure backend access. Enable system-assigned identity for simple scenarios, create user-assigned identities for shared access across multiple resources, and use Workload Identity for Kubernetes-based deployments. Grant the minimum required role (`Storage Blob Data Contributor`) on the specific state storage account, not the entire subscription.
