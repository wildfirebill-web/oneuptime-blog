# How to Build a Landing Zone with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Landing Zone, Architecture, OpenTofu, Organizations, Control Tower, Security

Description: Learn how to build an AWS Landing Zone using OpenTofu with AWS Organizations, Service Control Policies, centralized logging, and security baselines for enterprise-grade multi-account governance.

## Overview

An AWS Landing Zone provides a pre-configured, secure multi-account environment. OpenTofu provisions the organizational structure, foundational accounts, Service Control Policies, and centralized logging without AWS Control Tower, giving full IaC control.

## Step 1: AWS Organizations Structure

```hcl
# main.tf - AWS Organizations setup
resource "aws_organizations_organization" "org" {
  aws_service_access_principals = [
    "cloudtrail.amazonaws.com",
    "config.amazonaws.com",
    "sso.amazonaws.com",
    "securityhub.amazonaws.com",
    "guardduty.amazonaws.com",
  ]

  feature_set = "ALL"

  enabled_policy_types = [
    "SERVICE_CONTROL_POLICY",
    "TAG_POLICY",
  ]
}

# Organizational Unit hierarchy
resource "aws_organizations_organizational_unit" "security" {
  name      = "Security"
  parent_id = aws_organizations_organization.org.roots[0].id
}

resource "aws_organizations_organizational_unit" "infrastructure" {
  name      = "Infrastructure"
  parent_id = aws_organizations_organization.org.roots[0].id
}

resource "aws_organizations_organizational_unit" "workloads" {
  name      = "Workloads"
  parent_id = aws_organizations_organization.org.roots[0].id
}

resource "aws_organizations_organizational_unit" "sandbox" {
  name      = "Sandbox"
  parent_id = aws_organizations_organization.org.roots[0].id
}
```

## Step 2: Service Control Policies

```hcl
# Deny root account usage
resource "aws_organizations_policy" "deny_root" {
  name    = "DenyRootAccountUsage"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid      = "DenyRootAccountUsage"
      Effect   = "Deny"
      Action   = ["*"]
      Resource = ["*"]
      Condition = {
        StringLike = {
          "aws:PrincipalArn" = ["arn:aws:iam::*:root"]
        }
      }
    }]
  })
}

# Restrict to approved AWS regions
resource "aws_organizations_policy" "restrict_regions" {
  name    = "RestrictRegions"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid      = "DenyNonApprovedRegions"
      Effect   = "Deny"
      NotAction = [
        "cloudfront:*", "iam:*", "route53:*", "support:*",
        "sts:*", "organizations:*"
      ]
      Resource = ["*"]
      Condition = {
        StringNotEquals = {
          "aws:RequestedRegion" = ["us-east-1", "us-west-2", "eu-west-1"]
        }
      }
    }]
  })
}

# Attach policies to OUs
resource "aws_organizations_policy_attachment" "deny_root_to_root" {
  policy_id = aws_organizations_policy.deny_root.id
  target_id = aws_organizations_organization.org.roots[0].id
}
```

## Step 3: Centralized Log Archive Account

```hcl
# Create Log Archive account
resource "aws_organizations_account" "log_archive" {
  name  = "log-archive"
  email = "aws-log-archive@example.com"
  parent_id = aws_organizations_organizational_unit.security.id

  # Prevent account deletion via IaC
  close_on_deletion = false
}

# CloudTrail in each account sending to central S3 bucket
resource "aws_cloudtrail" "org_trail" {
  name                          = "org-trail"
  s3_bucket_name                = aws_s3_bucket.log_archive.bucket
  is_organization_trail         = true  # Captures all accounts
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]
    }
  }
}
```

## Step 4: Security Baseline (GuardDuty + Security Hub)

```hcl
# Enable GuardDuty at organization level
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs { enable = true }
    kubernetes { audit_logs { enable = true } }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes { enable = true }
      }
    }
  }
}

# Delegate GuardDuty administration to Security account
resource "aws_guardduty_organization_admin_account" "security" {
  admin_account_id = aws_organizations_account.security_tooling.id
}

# Enable Security Hub
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "aws_foundational" {
  standards_arn = "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
  depends_on    = [aws_securityhub_account.main]
}
```

## Summary

An AWS Landing Zone built with OpenTofu establishes multi-account governance before any workloads are deployed. Service Control Policies enforce guardrails that even account administrators cannot bypass. Centralized CloudTrail logging with log file validation creates an immutable audit trail across all accounts. Delegating GuardDuty and Security Hub to a dedicated Security account enables centralized threat detection without requiring cross-account access from individual workload accounts.
