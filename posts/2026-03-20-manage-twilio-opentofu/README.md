# How to Manage Twilio Resources with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Twilio, SMS, Communication, Messaging

Description: Learn how to manage Twilio phone numbers, messaging services, and Verify services using OpenTofu for reproducible communications infrastructure.

## Introduction

The Twilio provider for OpenTofu manages phone numbers, messaging services, Verify services, and other Twilio resources. Managing these as code ensures consistent configuration and makes provisioning new phone numbers for different environments repeatable.

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
  username = var.twilio_account_sid
  password = var.twilio_auth_token
}
```

## Phone Number Management

```hcl
# Search for available numbers
data "twilio_phone_numbers_available_local" "us_numbers" {
  country_code = "US"
  area_code    = "415"
  sms_enabled  = true
  voice_enabled = true
}

# Purchase a phone number
resource "twilio_phone_numbers_incoming" "main" {
  account_sid    = var.twilio_account_sid
  area_code      = "415"
  friendly_name  = "prod-main-number"

  # Webhooks for incoming messages and calls
  sms_url           = "https://api.example.com/webhooks/sms"
  sms_method        = "POST"
  voice_url         = "https://api.example.com/webhooks/voice"
  voice_method      = "POST"
  status_callback   = "https://api.example.com/webhooks/status"
}
```

## Messaging Service

```hcl
resource "twilio_messaging_services" "notifications" {
  account_sid      = var.twilio_account_sid
  friendly_name    = "App Notifications"

  # Sticky sender ensures same number used per recipient
  sticky_sender        = true
  mms_converter        = true
  smart_encoding       = true
  validity_period      = 14400  # 4 hours

  status_callback = "https://api.example.com/webhooks/message-status"
}

# Add the phone number to the messaging service
resource "twilio_messaging_services_phone_numbers" "main" {
  account_sid = var.twilio_account_sid
  service_sid = twilio_messaging_services.notifications.sid
  phone_number_sid = twilio_phone_numbers_incoming.main.sid
}
```

## Verify Service for 2FA

```hcl
resource "twilio_verify_services" "app_verify" {
  account_sid   = var.twilio_account_sid
  friendly_name = "My App Verification"

  # Code length and expiry
  code_length      = 6

  # Enable SMS channel
  # (SMS is enabled by default, additional channels below)

  # Push notification channel
  push {
    apn_credential_sid = var.apn_credential_sid
    fcm_credential_sid = var.fcm_credential_sid
  }

  # TOTP support
  totp {
    issuer          = "MyApp"
    code_length     = 6
    time_step       = 30
    skew            = 1
  }

  mailer_sid  = var.sendgrid_mailer_sid
}
```

## Subaccount Management

```hcl
# Create separate subaccounts per environment
resource "twilio_accounts_subaccounts" "staging" {
  friendly_name = "Staging Environment"
}

resource "twilio_accounts_subaccounts" "production" {
  friendly_name = "Production Environment"
}

# Outputs for use in application configuration
output "staging_account_sid" {
  value = twilio_accounts_subaccounts.staging.sid
}
```

## Outputs for Application Use

```hcl
output "messaging_service_sid" {
  value       = twilio_messaging_services.notifications.sid
  description = "Twilio Messaging Service SID for application configuration"
}

output "verify_service_sid" {
  value       = twilio_verify_services.app_verify.sid
  description = "Twilio Verify Service SID for 2FA implementation"
}

output "phone_number" {
  value       = twilio_phone_numbers_incoming.main.phone_number
  description = "The provisioned phone number in E.164 format"
}
```

## Storing Twilio Config in Parameter Store

```hcl
resource "aws_ssm_parameter" "twilio_messaging_sid" {
  name  = "/${var.environment}/twilio/messaging-service-sid"
  type  = "SecureString"
  value = twilio_messaging_services.notifications.sid
}

resource "aws_ssm_parameter" "twilio_verify_sid" {
  name  = "/${var.environment}/twilio/verify-service-sid"
  type  = "SecureString"
  value = twilio_verify_services.app_verify.sid
}
```

## Conclusion

Managing Twilio resources with OpenTofu ensures consistent messaging infrastructure across environments. Separate subaccounts for staging and production prevent test messages from affecting production usage limits. Storing Twilio SIDs in SSM Parameter Store makes them available to applications without hard-coding them in configuration files.
