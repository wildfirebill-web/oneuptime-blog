# How to Use tofu.applying for Plan vs Apply Differentiation in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, tofu.applying, Plan, Apply, Expressions, Infrastructure as Code, DevOps

Description: A guide to using the tofu.applying built-in value to differentiate between plan and apply phases in OpenTofu expressions.

## Introduction

OpenTofu provides a built-in value `tofu.applying` that evaluates to `true` during an apply operation and `false` during a plan operation. This allows you to write expressions that behave differently depending on whether you are previewing changes or actually applying them.

## Basic tofu.applying Usage

```hcl
# tofu.applying is true during apply, false during plan
output "current_phase" {
  value = tofu.applying ? "apply" : "plan"
}

# Use in locals for phase-specific configuration
locals {
  # Use a timestamp only during apply (stable for plans)
  deployment_id = tofu.applying ? timestamp() : "plan-preview"
}
```

## Ephemeral Values with tofu.applying

```hcl
# Fetch credentials only during apply, use placeholder during plan
ephemeral "aws_secretsmanager_secret_version" "api_key" {
  secret_id = "myapp/api-key"
}

locals {
  # During plan: use a placeholder to avoid unnecessary secret reads
  # During apply: use the actual secret value
  api_key = tofu.applying ? (
    ephemeral.aws_secretsmanager_secret_version.api_key.secret_string
  ) : "plan-time-placeholder"
}
```

## Timestamp Handling

```hcl
# timestamp() changes every plan, causing perpetual diffs
# Use tofu.applying to only record timestamps during apply

resource "terraform_data" "deployment_record" {
  input = {
    version     = var.app_version
    environment = var.environment
    # Only set actual timestamp during apply
    deployed_at = tofu.applying ? timestamp() : null
  }
}

output "deployment_info" {
  value = terraform_data.deployment_record.output
}
```

## Conditional Provisioner Arguments

```hcl
resource "terraform_data" "setup" {
  triggers_replace = var.app_version

  # Use tofu.applying to set realistic values only during apply
  input = {
    run_time = tofu.applying ? timestamp() : "(will be set during apply)"
    version  = var.app_version
  }

  provisioner "local-exec" {
    command = tofu.applying ? (
      "echo 'Deploying version ${var.app_version}'"
    ) : "true"  # No-op during plan
  }
}
```

## Notifications During Apply

```hcl
# Send notification only during actual apply, not during plan
resource "terraform_data" "deploy_notification" {
  triggers_replace = var.app_version

  provisioner "local-exec" {
    command = tofu.applying ? <<-EOT
      curl -X POST \
        -H "Content-Type: application/json" \
        -d '{"text": "Deploying ${var.app_name} v${var.app_version}"}' \
        ${var.slack_webhook_url}
    EOT : "echo 'Plan phase - no notification'"
  }
}
```

## Using with Ephemeral Resources

```hcl
# Some ephemeral resources are expensive to create
# Use tofu.applying to avoid creating them during plan

variable "generate_certificate" {
  type    = bool
  default = true
}

ephemeral "tls_private_key" "app" {
  # Only generate the key during actual apply
  enabled   = tofu.applying && var.generate_certificate
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_acm_certificate" "app" {
  enabled         = var.generate_certificate
  private_key     = tofu.applying ? ephemeral.tls_private_key.app.private_key_pem : null
  certificate_body = tofu.applying ? var.certificate_body : null
}
```

## Plan-Safe Outputs

```hcl
# Some outputs may reference ephemeral values
# Make them plan-safe with tofu.applying

ephemeral "vault_generic_secret" "app_token" {
  path = "secret/myapp/token"
}

output "app_token_preview" {
  value = tofu.applying ? (
    "Token fetched (${length(ephemeral.vault_generic_secret.app_token.data["token"])} chars)"
  ) : "(available after apply)"
  description = "Summary of app token status"
}
```

## Combining with Conditionals

```hcl
locals {
  # Build configuration that varies by phase
  operation_config = {
    phase       = tofu.applying ? "apply" : "plan"
    timestamp   = tofu.applying ? timestamp() : null
    dry_run     = !tofu.applying

    # Fetch real values only during apply
    api_endpoint = tofu.applying ? (
      "https://api.${var.domain}/v1"
    ) : "https://api.example.com/v1"
  }
}

resource "terraform_data" "operation_log" {
  input = local.operation_config
}
```

## Debugging Phase Differences

```hcl
# Add phase indicator to help debug plan vs apply differences
output "debug_phase" {
  value = {
    phase      = tofu.applying ? "APPLY" : "PLAN"
    is_applying = tofu.applying
  }
}

# In practice, this helps understand why values differ between plan and apply
resource "aws_ssm_parameter" "deployment_phase" {
  name  = "/myapp/last-operation-phase"
  type  = "String"
  value = tofu.applying ? "apply-${timestamp()}" : "plan-preview"
}
```

## Conclusion

The `tofu.applying` built-in value enables you to write configurations that behave intelligently based on the current operation phase. It is particularly useful when working with ephemeral resources (to avoid unnecessary credential fetches during plan), timestamps (to prevent perpetual diffs), notifications (to only alert during actual deployments), and debugging (to understand what values will look like during apply vs plan). Use `tofu.applying` to make your configurations more efficient and to avoid side effects during the planning phase.
