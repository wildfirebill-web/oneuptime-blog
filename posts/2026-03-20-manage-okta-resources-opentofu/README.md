# How to Manage Okta Resources with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Okta, Identity, SSO, User Management

Description: Learn how to manage Okta applications, groups, users, and policies using OpenTofu to automate identity and access management as code.

## Introduction

The Okta provider for OpenTofu lets you manage your Okta organization's applications, groups, policies, and user assignments as code. This enables consistent SSO configuration across environments and makes access management auditable through version control.

## Provider Configuration

```hcl
terraform {
  required_providers {
    okta = {
      source  = "okta/okta"
      version = "~> 4.0"
    }
  }
}

provider "okta" {
  org_name  = var.okta_org_name   # e.g., "mycompany"
  base_url  = "okta.com"
  api_token = var.okta_api_token  # From OKTA_API_TOKEN env var
}
```

## Managing Groups

```hcl
resource "okta_group" "engineering" {
  name        = "Engineering"
  description = "All engineering team members"
}

resource "okta_group" "platform_team" {
  name        = "Platform Team"
  description = "Platform engineering team with infrastructure access"
}

# Group rule - automatically assign users based on profile attributes
resource "okta_group_rule" "engineers_rule" {
  name              = "Auto-assign engineers"
  status            = "ACTIVE"
  group_assignments = [okta_group.engineering.id]
  expression_type   = "urn:okta:expression:1.0"
  expression_value  = "user.department == \"Engineering\""
}
```

## Creating an OIDC Application

```hcl
resource "okta_app_oauth" "internal_app" {
  label                     = "Internal Platform App"
  type                      = "web"
  grant_types               = ["authorization_code", "refresh_token"]
  redirect_uris             = ["https://app.example.com/callback"]
  post_logout_redirect_uris = ["https://app.example.com/logout"]
  response_types            = ["code"]
  token_endpoint_auth_method = "client_secret_basic"

  lifecycle {
    ignore_changes = [client_secret]
  }
}

# Assign groups to the application
resource "okta_app_group_assignments" "internal_app" {
  app_id = okta_app_oauth.internal_app.id

  group {
    id = okta_group.engineering.id
  }

  group {
    id = okta_group.platform_team.id
  }
}
```

## SAML Application for AWS SSO

```hcl
resource "okta_app_saml" "aws_sso" {
  label             = "AWS SSO"
  sso_url           = "https://signin.aws.amazon.com/saml"
  recipient         = "https://signin.aws.amazon.com/saml"
  destination       = "https://signin.aws.amazon.com/saml"
  audience          = "https://signin.aws.amazon.com/saml"
  subject_name_id_template = "$${user.login}"
  subject_name_id_format   = "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress"
  response_signed   = true
  signature_algorithm = "RSA_SHA256"
  digest_algorithm  = "SHA256"
  honor_force_authn = false
  authn_context_class_ref = "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport"

  attribute_statements {
    name      = "https://aws.amazon.com/SAML/Attributes/Role"
    namespace = "urn:oasis:names:tc:SAML:2.0:attrname-format:uri"
    type      = "EXPRESSION"
    values    = ["appuser.awsRole"]
  }
}
```

## Password Policy Management

```hcl
resource "okta_policy_password" "strong" {
  name            = "Strong Password Policy"
  status          = "ACTIVE"
  priority        = 1
  groups_included = [okta_group.engineering.id]

  password_min_length            = 14
  password_min_lowercase         = 1
  password_min_uppercase         = 1
  password_min_number            = 1
  password_min_symbol            = 1
  password_exclude_username      = true
  password_dictionary_lookup     = true
  password_max_age_days          = 90
  password_history_count         = 10

  recovery_email_token = 60
}
```

## MFA Enrollment Policy

```hcl
resource "okta_policy_mfa" "required_mfa" {
  name     = "Require MFA for Engineering"
  status   = "ACTIVE"
  priority = 1

  groups_included = [okta_group.engineering.id]

  okta_totp {
    enroll = "REQUIRED"
  }

  fido_webauthn {
    enroll = "OPTIONAL"
  }
}
```

## Outputs for Reference

```hcl
output "app_client_id" {
  value       = okta_app_oauth.internal_app.client_id
  description = "OIDC client ID for the internal app"
}

output "okta_issuer_url" {
  value = "https://${var.okta_org_name}.okta.com/oauth2/default"
}
```

## Conclusion

Managing Okta with OpenTofu brings the same benefits as infrastructure-as-code to identity management: consistent group structures, auditable application configurations, and reproducible policy enforcement. Critical SSO configurations that previously required UI clicks or manual API calls can now go through pull request reviews before taking effect.
