# How to Set Up Azure App Service Authentication with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Azure, App Service, Authentication, OpenTofu, OAuth2, Azure AD

Description: Learn how to configure Azure App Service built-in authentication (Easy Auth) with OpenTofu to add identity provider authentication without changing application code.

## Overview

Azure App Service built-in authentication (Easy Auth) provides a plug-in authentication middleware that integrates with Azure AD, Google, Facebook, Twitter, and GitHub. OpenTofu configures the authentication settings declaratively.

## Step 1: Create Web App with Authentication

```hcl
# main.tf - Web App with Easy Auth enabled

resource "azurerm_linux_web_app" "app" {
  name                = "my-authenticated-app"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.plan.id
  https_only          = true

  site_config {
    application_stack {
      node_version = "20-lts"
    }
  }
}
```

## Step 2: Azure AD (Microsoft) Authentication

```hcl
# Register an app in Azure AD for authentication
resource "azuread_application" "app_auth" {
  display_name     = "MyAppAuthentication"
  sign_in_audience = "AzureADMyOrg"

  web {
    redirect_uris = [
      "https://${azurerm_linux_web_app.app.default_hostname}/.auth/login/aad/callback"
    ]
  }
}

resource "azuread_application_password" "app_auth_secret" {
  application_id = azuread_application.app_auth.id
  display_name   = "Easy Auth Secret"
  end_date       = "2027-01-01T00:00:00Z"
}

# Configure Easy Auth with Azure AD
resource "azurerm_linux_web_app_auth_settings_v2" "app_auth" {
  app_id = azurerm_linux_web_app.app.id

  # Require authentication for all requests
  auth_enabled           = true
  require_authentication = true
  unauthenticated_action = "RedirectToLoginPage"

  # Login settings
  login {
    token_store_enabled = true
    token_refresh_extension_hours = 72
  }

  # Azure Active Directory provider
  active_directory_v2 {
    client_id                  = azuread_application.app_auth.client_id
    tenant_auth_endpoint       = "https://login.microsoftonline.com/${data.azuread_client_config.current.tenant_id}/v2.0"
    client_secret_setting_name = "MICROSOFT_PROVIDER_AUTHENTICATION_SECRET"
  }

  # Set the client secret as an app setting
  # (use Key Vault reference in production)
}

# App setting to hold the OAuth client secret
resource "azurerm_linux_web_app" "app_with_secret" {
  # ...existing config...
  app_settings = {
    MICROSOFT_PROVIDER_AUTHENTICATION_SECRET = azuread_application_password.app_auth_secret.value
  }
}
```

## Step 3: Multi-Provider Authentication

```hcl
# Allow both Azure AD and GitHub authentication
resource "azurerm_linux_web_app_auth_settings_v2" "multi_provider_auth" {
  app_id = azurerm_linux_web_app.app.id

  auth_enabled           = true
  require_authentication = true
  unauthenticated_action = "Return401"  # Return 401 for API apps

  login {
    token_store_enabled = true
  }

  active_directory_v2 {
    client_id            = azuread_application.app_auth.client_id
    tenant_auth_endpoint = "https://login.microsoftonline.com/${data.azuread_client_config.current.tenant_id}/v2.0"
    client_secret_setting_name = "AAD_CLIENT_SECRET"
  }

  github_v2 {
    client_id                  = var.github_client_id
    client_secret_setting_name = "GITHUB_CLIENT_SECRET"
    scopes                     = ["read:user", "user:email"]
  }
}
```

## Step 4: Outputs

```hcl
output "auth_callback_url" {
  value = "https://${azurerm_linux_web_app.app.default_hostname}/.auth/login/aad/callback"
}

output "app_client_id" {
  value = azuread_application.app_auth.client_id
}
```

## Summary

Azure App Service Easy Auth configured with OpenTofu adds authentication to any web app without code changes. The `auth_settings_v2` resource supports modern OAuth2 providers and token store for session management, making it ideal for quickly securing internal tools and APIs.
