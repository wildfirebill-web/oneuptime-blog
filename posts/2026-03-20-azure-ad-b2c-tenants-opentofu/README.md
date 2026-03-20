# How to Create Azure AD B2C Tenants with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Azure, Azure AD B2C, Identity, Authentication, Infrastructure as Code

Description: Learn how to provision Azure Active Directory B2C tenants and configure them for customer identity and access management using OpenTofu.

## Introduction

Azure AD B2C is a customer identity and access management (CIAM) service that lets you customize and control how customers sign up, sign in, and manage their profiles. OpenTofu manages B2C tenant creation and configuration as code.

## Provider Configuration

B2C resources require both the standard AzureRM provider and the AzureAD provider.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.0"
    }
  }
}
```

## Creating the B2C Tenant Resource

```hcl
resource "azurerm_aadb2c_directory" "main" {
  country_code            = "US"
  data_residency_location = "United States"
  display_name            = "${var.app_name} Customer Identity"
  domain_name             = "${var.app_name}.onmicrosoft.com"
  resource_group_name     = azurerm_resource_group.main.name

  # SKU: PremiumP1 or PremiumP2 for advanced features
  sku_name = "PremiumP1"

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Configuring the AzureAD Provider for B2C

After the tenant is created, configure a second provider instance targeting the B2C tenant.

```hcl
provider "azuread" {
  alias     = "b2c"
  tenant_id = azurerm_aadb2c_directory.main.tenant_id
}
```

## Creating Application Registrations in B2C

```hcl
resource "azuread_application" "web_app" {
  provider     = azuread.b2c
  display_name = "${var.app_name} Web Application"

  web {
    redirect_uris = ["https://app.example.com/auth/callback"]
    implicit_grant {
      access_token_issuance_enabled = false
      id_token_issuance_enabled     = true
    }
  }

  required_resource_access {
    resource_app_id = "00000003-0000-0000-c000-000000000000" # MS Graph

    resource_access {
      id   = "37f7f235-527c-4136-accd-4a02d197296e" # openid
      type = "Scope"
    }
  }
}
```

## Creating B2C User Flow Policies

User flows define the sign-up/sign-in experience. These are typically configured via the Azure portal or MS Graph API, but you can reference them in outputs.

```hcl
output "tenant_id" {
  value = azurerm_aadb2c_directory.main.tenant_id
}

output "b2c_domain" {
  value = azurerm_aadb2c_directory.main.domain_name
}

output "web_app_client_id" {
  value = azuread_application.web_app.application_id
}
```

## Variables

```hcl
variable "app_name"    { type = string }
variable "environment" { type = string }

resource "azurerm_resource_group" "main" {
  name     = "rg-b2c-${var.environment}"
  location = "East US"
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

Azure AD B2C tenant provisioning with OpenTofu automates the creation of customer identity directories and application registrations. While some B2C features (user flows, custom policies) require portal or API configuration, the core infrastructure can be fully managed as code.
