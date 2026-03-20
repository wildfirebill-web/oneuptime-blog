# How to Authenticate with Azure Using Service Principal in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Service Principal, Authentication, CI/CD

Description: Learn how to create an Azure Service Principal and configure OpenTofu to authenticate using client secret or certificate-based credentials for CI/CD pipelines.

## Introduction

An Azure Service Principal is a non-human identity used by applications, services, and automation tools to access Azure resources. It is the recommended authentication method for CI/CD pipelines running OpenTofu.

## Creating a Service Principal

```bash
# Create a service principal with Contributor role on a specific subscription

az ad sp create-for-rbac \
  --name "opentofu-deploy-sp" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json
```

This outputs:
```json
{
  "appId": "CLIENT_ID",
  "password": "CLIENT_SECRET",
  "tenant": "TENANT_ID"
}
```

## Client Secret Authentication

```hcl
provider "azurerm" {
  features {}

  # Credentials from environment variables (preferred)
  # ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID
  # Or configure directly (not recommended for production)
  client_id       = var.client_id
  client_secret   = var.client_secret
  tenant_id       = var.tenant_id
  subscription_id = var.subscription_id
}
```

## Certificate-Based Authentication

```hcl
# Generate a certificate for the service principal
resource "tls_private_key" "sp" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "tls_self_signed_cert" "sp" {
  private_key_pem = tls_private_key.sp.private_key_pem

  subject {
    common_name  = "opentofu-service-principal"
    organization = "My Organisation"
  }

  validity_period_hours = 8760  # 1 year
  allowed_uses          = ["cert_signing", "key_encipherment", "digital_signature"]
}

# Upload the certificate to the service principal
resource "azuread_application_certificate" "sp" {
  application_id = azuread_application.opentofu.id
  type           = "AsymmetricX509Cert"
  value          = tls_self_signed_cert.sp.cert_pem
  end_date       = timeadd(timestamp(), "8760h")
}
```

## GitHub Actions with Service Principal

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      ARM_CLIENT_ID:       ${{ secrets.AZURE_CLIENT_ID }}
      ARM_CLIENT_SECRET:   ${{ secrets.AZURE_CLIENT_SECRET }}
      ARM_TENANT_ID:       ${{ secrets.AZURE_TENANT_ID }}
      ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
      - run: tofu init
      - run: tofu apply -auto-approve
```

## Least-Privilege Role Assignment

Instead of Contributor on the entire subscription, scope permissions to specific resource groups:

```hcl
resource "azurerm_role_assignment" "opentofu_rg" {
  scope                = azurerm_resource_group.app.id
  role_definition_name = "Contributor"
  principal_id         = azuread_service_principal.opentofu.object_id
}
```

## Conclusion

Service Principals are the standard CI/CD authentication method for Azure. Prefer certificate authentication over client secrets for production as certificates do not expire silently. Always scope the role assignment to the minimum required resource group or subscription, not the entire Azure AD tenant.
