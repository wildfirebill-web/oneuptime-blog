# How to Design a Compliance Module for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Compliance, AWS Config, Security Hub, Module, Governance

Description: Learn how to design a compliance module for OpenTofu that enables AWS Config rules, Security Hub standards, and CloudTrail logging to enforce regulatory requirements.

## Introduction

Compliance infrastructure needs to be consistent across all accounts and environments. A compliance module encapsulates AWS Config rule sets, Security Hub standards, CloudTrail configuration, and AWS Organizations policy enforcement - making compliance-as-code reproducible.

## variables.tf

```hcl
variable "environment"       { type = string }
variable "enable_config"     { type = bool; default = true }
variable "enable_security_hub" { type = bool; default = true }
variable "enable_guardduty"  { type = bool; default = true }
variable "enable_cloudtrail" { type = bool; default = true }

variable "cloudtrail_s3_bucket" { type = string; default = "" }

variable "config_rules" {
  description = "AWS Config managed rules to enable"
  type = map(object({
    source_identifier = string
    input_parameters  = optional(map(string), {})
  }))
  default = {
    "encrypted-volumes" = {
      source_identifier = "ENCRYPTED_VOLUMES"
    }
    "s3-bucket-public-read-prohibited" = {
      source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
    }
    "mfa-enabled-for-iam-console" = {
      source_identifier = "MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS"
    }
    "root-account-mfa" = {
      source_identifier = "ROOT_ACCOUNT_MFA_ENABLED"
    }
  }
}

variable "security_hub_standards" {
  type    = list(string)
  default = [
    "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.2.0",
    "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
  ]
}

variable "tags" { type = map(string); default = {} }
```

## main.tf

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

locals {
  tags = merge({ Environment = var.environment, ManagedBy = "OpenTofu" }, var.tags)
}

# AWS Config Recorder

resource "aws_config_configuration_recorder" "main" {
  count    = var.enable_config ? 1 : 0
  name     = "default"
  role_arn = aws_iam_role.config[0].arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

resource "aws_iam_role" "config" {
  count = var.enable_config ? 1 : 0
  name  = "aws-config-role"
  assume_role_policy = jsonencode({
    Version   = "2012-10-17"
    Statement = [{ Effect = "Allow"; Principal = { Service = "config.amazonaws.com" }; Action = "sts:AssumeRole" }]
  })
}

resource "aws_iam_role_policy_attachment" "config" {
  count      = var.enable_config ? 1 : 0
  role       = aws_iam_role.config[0].name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"
}

resource "aws_config_config_rule" "rules" {
  for_each = var.enable_config ? var.config_rules : {}

  name = each.key
  source {
    owner             = "AWS"
    source_identifier = each.value.source_identifier
  }

  dynamic "input_parameters" {
    for_each = length(each.value.input_parameters) > 0 ? [each.value.input_parameters] : []
    content {}
  }

  depends_on = [aws_config_configuration_recorder.main]
}

# AWS Security Hub
resource "aws_securityhub_account" "main" {
  count = var.enable_security_hub ? 1 : 0
}

resource "aws_securityhub_standards_subscription" "standards" {
  for_each      = var.enable_security_hub ? toset(var.security_hub_standards) : toset([])
  standards_arn = each.key
  depends_on    = [aws_securityhub_account.main]
}

# GuardDuty
resource "aws_guardduty_detector" "main" {
  count  = var.enable_guardduty ? 1 : 0
  enable = true
  tags   = local.tags
}

# CloudTrail
resource "aws_cloudtrail" "main" {
  count                         = var.enable_cloudtrail && var.cloudtrail_s3_bucket != "" ? 1 : 0
  name                          = "compliance-trail"
  s3_bucket_name                = var.cloudtrail_s3_bucket
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  tags                          = local.tags
}
```

## outputs.tf

```hcl
output "config_enabled"        { value = var.enable_config }
output "security_hub_enabled"  { value = var.enable_security_hub }
output "guardduty_detector_id" { value = var.enable_guardduty ? aws_guardduty_detector.main[0].id : null }
output "config_rule_arns" {
  value = { for k, rule in aws_config_config_rule.rules : k => rule.arn }
}
```

## Conclusion

This compliance module applies a standard security baseline to any AWS account by enabling AWS Config rules, Security Hub standards, GuardDuty threat detection, and CloudTrail audit logging. Applying it as part of account provisioning ensures every account starts compliant with organizational requirements.
