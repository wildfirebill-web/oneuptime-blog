# How to Manage Twilio Resources with OpenTofu - Resources

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Twilio, SMS, Infrastructure as Code, Communications API

Description: Learn how to manage Twilio phone numbers, messaging services, and API keys using OpenTofu and the official Twilio provider.

## Introduction

Twilio provides cloud communications APIs for SMS, voice, video, and email. Managing Twilio resources through OpenTofu allows teams to provision phone numbers, configure messaging services, and manage credentials as version-controlled infrastructure.

## Prerequisites

- OpenTofu installed (v1.6+)
- A Twilio account
- Twilio Account SID and Auth Token

## Provider Configuration

```hcl
terraform {
  required_providers {
    twilio = {
      source  = "twilio/twilio"
      version = "~> 0.18"
    }
  }
}

provider "twilio" {
  account_sid  = var.twilio_account_sid
  auth_token   = var.twilio_auth_token
}
```

```bash
export TWILIO_ACCOUNT_SID="ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export TWILIO_AUTH_TOKEN="your-auth-token"
```

## Purchasing a Phone Number

```hcl
resource "twilio_phone_numbers_toll_free" "sms_number" {
  area_code = "800"
}

# Or search for a specific number

data "twilio_phone_numbers_available_toll_free_numbers" "available" {
  country_code = "US"
  sms_enabled  = true
  mms_enabled  = true
}
```

## Creating a Messaging Service

```hcl
resource "twilio_messaging_service" "notifications" {
  friendly_name = "Notifications Service"

  inbound_request_url = "https://api.example.com/webhooks/twilio/inbound"
  status_callback_url = "https://api.example.com/webhooks/twilio/status"

  fallback_url        = "https://api.example.com/webhooks/twilio/fallback"
  fallback_method     = "POST"
}
```

## Adding Phone Numbers to a Messaging Service

```hcl
resource "twilio_messaging_service_phone_number" "main" {
  service_sid  = twilio_messaging_service.notifications.sid
  phone_number_sid = twilio_phone_numbers_toll_free.sms_number.sid
}
```

## Managing API Keys

Create scoped API keys for application access (avoid using Account SID/Auth Token in production):

```hcl
resource "twilio_api_accounts_keys" "app_service" {
  friendly_name = "app-notifications-service"
}

output "api_key_sid" {
  value = twilio_api_accounts_keys.app_service.sid
}

output "api_key_secret" {
  value     = twilio_api_accounts_keys.app_service.secret
  sensitive = true
}
```

## Configuring a Verify Service

```hcl
resource "twilio_verify_services" "user_verification" {
  friendly_name         = "User Verification"
  code_length           = 6
  lookup_enabled        = true
  skip_sms_to_landlines = true
  dtmf_input_required   = true
  tts_name              = "My App"
  psd2_enabled          = false
  do_not_share_warning_enabled = true
}

output "verify_service_sid" {
  value = twilio_verify_services.user_verification.sid
}
```

## Subaccounts

Manage multiple clients or environments with subaccounts:

```hcl
resource "twilio_api_accounts" "staging" {
  friendly_name = "staging-environment"
}

resource "twilio_api_accounts" "client_a" {
  friendly_name = "client-a-production"
}
```

## Storing Credentials Securely

```hcl
resource "aws_secretsmanager_secret" "twilio_api_key" {
  name = "/app/twilio/api-key"
}

resource "aws_secretsmanager_secret_version" "twilio_api_key" {
  secret_id = aws_secretsmanager_secret.twilio_api_key.id
  secret_string = jsonencode({
    sid    = twilio_api_accounts_keys.app_service.sid
    secret = twilio_api_accounts_keys.app_service.secret
  })
}
```

## Best Practices

- Use API keys instead of Account SID/Auth Token for application authentication.
- Separate messaging services by use case (notifications, marketing, alerts).
- Add multiple phone numbers to a messaging service for higher throughput and geographic redundancy.
- Enable `lookup_enabled` on Verify services to avoid sending to non-mobile numbers.
- Use subaccounts to isolate production and staging environments.

## Conclusion

OpenTofu's Twilio provider enables consistent, version-controlled management of your communications infrastructure. Phone numbers, messaging services, and credentials are all managed as code, making it easy to replicate setups across environments and audit all configuration changes.
