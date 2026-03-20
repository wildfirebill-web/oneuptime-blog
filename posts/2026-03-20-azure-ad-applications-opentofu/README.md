# How to Set Up Azure AD Applications with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, Azure AD, Applications, OpenTofu, Identity, OAuth2

Description: Learn how to create and configure Azure Active Directory applications with OpenTofu for OAuth2/OIDC authentication, API permissions, and service principal setup.

## Overview

Azure AD Applications (now Microsoft Entra ID App Registrations) are the foundation for authenticating applications and services with Azure. OpenTofu's AzureAD provider lets you define app registrations, service principals, and credentials as code.

## Step 1: Configure the AzureAD Provider

```hcl
# providers.tf
terraform {
  required_providers {
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.47"
    }
  }
}

provider "azuread" {}

data "azuread_client_config" "current" {}
```

## Step 2: Create an App Registration

```hcl
# main.tf - App registration for a web application
resource "azuread_application" "web_app" {
  display_name     = "MyWebApplication"
  identifier_uris  = ["api://mywebapp"]
  sign_in_audience = "AzureADMyOrg"  # Single tenant

  # Configure as a web application with redirect URIs
  web {
    redirect_uris = [
      "https://myapp.example.com/auth/callback",
      "https://localhost:3000/auth/callback",  # Development
    ]
    logout_url = "https://myapp.example.com/auth/logout"

    implicit_grant {
      access_token_issuance_enabled = false
      id_token_issuance_enabled     = true
    }
  }

  # Define the app's own API scopes
  api {
    mapped_claims_enabled          = true
    requested_access_token_version = 2

    oauth2_permission_scope {
      admin_consent_description  = "Allow read access to user data"
      admin_consent_display_name = "Read user data"
      enabled                    = true
      id                         = "00000000-0000-0000-0000-000000000001"
      type                       = "User"
      user_consent_description   = "Allow the app to read your data"
      user_consent_display_name  = "Read your data"
      value                      = "data.read"
    }
  }

  # Request permissions to call Microsoft Graph
  required_resource_access {
    resource_app_id = "00000003-0000-0000-c000-000000000000"  # Microsoft Graph

    resource_access {
      id   = "e1fe6dd8-ba31-4d61-89e7-88639da4683d"  # User.Read
      type = "Scope"
    }
  }
}
```

## Step 3: Create a Service Principal

```hcl
# Service principal enables the app to authenticate programmatically
resource "azuread_service_principal" "web_app_sp" {
  client_id                    = azuread_application.web_app.client_id
  app_role_assignment_required = false

  tags = ["webapp", "production"]
}
```

## Step 4: Create Client Secret

```hcl
# Client secret for server-to-server authentication
resource "azuread_application_password" "web_app_secret" {
  application_id = azuread_application.web_app.id
  display_name   = "Production Client Secret"
  end_date       = "2027-01-01T00:00:00Z"
}

# Store the secret in Key Vault
resource "azurerm_key_vault_secret" "app_client_secret" {
  name         = "webapp-client-secret"
  value        = azuread_application_password.web_app_secret.value
  key_vault_id = azurerm_key_vault.kv.id
}
```

## Step 5: Outputs

```hcl
output "application_client_id" {
  value       = azuread_application.web_app.client_id
  description = "App registration client ID (also called Application ID)"
}

output "tenant_id" {
  value       = data.azuread_client_config.current.tenant_id
  description = "Azure AD tenant ID"
}

output "service_principal_object_id" {
  value       = azuread_service_principal.web_app_sp.object_id
}
```

## Summary

Azure AD Application registrations managed with OpenTofu provide a repeatable way to set up OAuth2/OIDC authentication for applications. By storing client secrets in Key Vault and defining API permissions as code, you ensure consistent configuration across environments and reduce manual portal configuration errors.
