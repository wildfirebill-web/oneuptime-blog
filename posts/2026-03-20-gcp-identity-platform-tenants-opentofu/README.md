# How to Create GCP Identity Platform Tenants with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GCP, Identity Platform, Firebase Auth, Multi-Tenancy, Infrastructure as Code

Description: Learn how to create and configure GCP Identity Platform tenants for multi-tenant authentication using OpenTofu.

## Introduction

GCP Identity Platform (Firebase Authentication) supports multi-tenancy, allowing you to create isolated authentication environments for different customers or applications. OpenTofu manages tenant creation and configuration as code.

## Enabling Identity Platform

First, ensure the Identity Toolkit API is enabled in your project.

```hcl
resource "google_project_service" "identity_toolkit" {
  project = var.project_id
  service = "identitytoolkit.googleapis.com"
}
```

## Creating a Tenant

```hcl
resource "google_identity_platform_tenant" "customer_a" {
  project      = var.project_id
  display_name = "Customer A"

  # Allow email/password sign-in
  allow_password_signup = true

  # Disable legacy sign-in methods
  enable_email_link_signin = false

  # MFA configuration (requires Identity Platform billing plan)
  mfa_config {
    state = "ENABLED"
    enabled_providers = ["PHONE_SMS"]
  }
}
```

## Configuring OAuth Identity Providers for a Tenant

```hcl
resource "google_identity_platform_tenant_oauth_idp_config" "google_oidc" {
  project   = var.project_id
  tenant    = google_identity_platform_tenant.customer_a.name
  name      = "oidc.google-workspace"
  client_id = var.google_client_id
  issuer    = "https://accounts.google.com"
  enabled   = true

  client_secret = var.google_client_secret
}
```

## Configuring SAML for a Tenant

```hcl
resource "google_identity_platform_tenant_saml_idp_config" "okta_saml" {
  project   = var.project_id
  tenant    = google_identity_platform_tenant.customer_a.name
  name      = "saml.okta"

  idp_config {
    idp_entity_id = "http://www.okta.com/${var.okta_app_id}"
    sign_request  = true

    sso_url = "https://${var.okta_domain}/app/${var.okta_app_id}/sso/saml"

    idp_certificates {
      x509_certificate = var.okta_x509_certificate
    }
  }

  sp_config {
    sp_entity_id = "https://myapp.example.com"
    callback_uri = "https://myapp.example.com/__/auth/handler"
  }

  enabled = true
}
```

## Tenant Default Config

```hcl
resource "google_identity_platform_config" "default" {
  project = var.project_id

  # Multi-tenancy must be enabled at the project level
  multi_tenant {
    allow_tenants         = true
    default_tenant_location = "projects/${var.project_id}"
  }

  sign_in {
    allow_duplicate_emails = false

    anonymous {
      enabled = false
    }

    email {
      enabled           = true
      password_required = true
    }
  }
}
```

## Variables and Outputs

```hcl
variable "project_id"          { type = string }
variable "google_client_id"    { type = string }
variable "google_client_secret"{ type = string sensitive = true }
variable "okta_domain"         { type = string }
variable "okta_app_id"         { type = string }
variable "okta_x509_certificate"{ type = string sensitive = true }

output "tenant_id" {
  description = "Tenant ID to pass to Firebase SDK"
  value       = google_identity_platform_tenant.customer_a.name
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

GCP Identity Platform multi-tenancy enables isolated authentication for each of your customers. OpenTofu manages tenant creation, OAuth and SAML provider configurations, and project-level settings, making multi-tenant identity infrastructure reproducible and auditable.
