# How to Configure Azure Backend with Service Principal Authentication in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backends, Azure

Description: Learn how to configure Service Principal authentication for the OpenTofu Azure Blob Storage backend for CI/CD and non-interactive deployments.

## Introduction

Service Principal authentication is the standard approach for CI/CD pipelines accessing the Azure Blob Storage backend. A Service Principal is an application identity in Azure Active Directory with specific permissions — it enables non-interactive, automated access to Azure resources.

## Creating a Service Principal

```bash
# Create a service principal and get credentials
az ad sp create-for-rbac \
  --name "opentofu-state-sp" \
  --role "Storage Blob Data Contributor" \
  --scopes "/subscriptions/SUB_ID/resourceGroups/terraform-state-rg/providers/Microsoft.Storage/storageAccounts/acmecorptofu"

# Output:
# {
#   "appId": "YOUR_CLIENT_ID",
#   "displayName": "opentofu-state-sp",
#   "password": "YOUR_CLIENT_SECRET",
#   "tenant": "YOUR_TENANT_ID"
# }
```

## Environment Variable Authentication

```bash
# Set credentials as environment variables
export ARM_CLIENT_ID="YOUR_CLIENT_ID"
export ARM_CLIENT_SECRET="YOUR_CLIENT_SECRET"
export ARM_SUBSCRIPTION_ID="YOUR_SUBSCRIPTION_ID"
export ARM_TENANT_ID="YOUR_TENANT_ID"

tofu init
```

## Backend Configuration

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "acmecorptofu"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
    # Credentials are provided via environment variables
  }
}
```

## Using a Client Certificate

For certificate-based authentication (more secure than client secret):

```hcl
terraform {
  backend "azurerm" {
    resource_group_name      = "terraform-state-rg"
    storage_account_name     = "acmecorptofu"
    container_name           = "tfstate"
    key                      = "production.terraform.tfstate"
    subscription_id          = "YOUR_SUBSCRIPTION_ID"
    tenant_id                = "YOUR_TENANT_ID"
    client_id                = "YOUR_CLIENT_ID"
    client_certificate_path  = "/path/to/certificate.pfx"
    client_certificate_password = var.cert_password
  }
}
```

## GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Init and Deploy
        env:
          ARM_CLIENT_ID: ${{ secrets.ARM_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.ARM_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.ARM_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.ARM_TENANT_ID }}
        run: |
          tofu init
          tofu plan
          tofu apply -auto-approve
```

## Rotating Service Principal Credentials

```bash
# Create a new client secret
az ad sp credential reset \
  --id "YOUR_APP_ID" \
  --display-name "opentofu-state-sp-new"

# Update the CI/CD secrets with new credentials
# Test before deleting old credentials

# Delete old credentials
az ad sp credential delete \
  --id "YOUR_APP_ID" \
  --key-id "OLD_KEY_ID"
```

## Partial Backend Configuration

Keep sensitive values out of source code:

```hcl
# backend.tf — committed to source control
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "acmecorptofu"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
    # subscription_id, tenant_id, client_id, client_secret via environment
  }
}
```

## Conclusion

Service Principal authentication is the standard approach for automated OpenTofu deployments on Azure. Create a Service Principal with minimal permissions (Storage Blob Data Contributor on the state account), store credentials in CI/CD secrets, and pass them as environment variables. Rotate credentials regularly and prefer certificate authentication over client secrets for improved security.
