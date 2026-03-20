# How to Set Up AWS GuardDuty with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, GuardDuty, Security, Threat Detection, Infrastructure as Code

Description: Learn how to enable and configure AWS GuardDuty with OpenTofu to continuously monitor for malicious activity and unauthorized behavior across your AWS accounts.

## Introduction

AWS GuardDuty is a managed threat detection service that analyzes CloudTrail events, VPC Flow Logs, and DNS logs to identify malicious activity like crypto mining, compromised instances, unauthorized access, and data exfiltration. It uses ML, anomaly detection, and integrated threat intelligence without requiring agents or additional log ingestion.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with GuardDuty and Organizations permissions

## Step 1: Enable GuardDuty Detector

```hcl
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs {
      enable = true  # Analyze S3 data plane events
    }
    kubernetes {
      audit_logs {
        enable = true  # Analyze EKS audit logs
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true  # Scan EBS volumes for malware
        }
      }
    }
  }

  finding_publishing_frequency = "SIX_HOURS"  # FIFTEEN_MINUTES, ONE_HOUR, or SIX_HOURS

  tags = {
    Name        = "${var.project_name}-guardduty"
    Environment = var.environment
  }
}
```

## Step 2: Add Trusted IP List (Whitelist)

```hcl
resource "aws_s3_bucket" "guardduty_lists" {
  bucket = "${var.project_name}-guardduty-lists-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_object" "trusted_ips" {
  bucket  = aws_s3_bucket.guardduty_lists.id
  key     = "trusted-ips.txt"
  content = join("\n", var.trusted_ip_ranges)  # e.g., ["10.0.0.0/8", "172.16.0.0/12"]
}

resource "aws_guardduty_ipset" "trusted" {
  activate    = true
  detector_id = aws_guardduty_detector.main.id
  format      = "TXT"
  location    = "s3://${aws_s3_bucket.guardduty_lists.id}/${aws_s3_object.trusted_ips.key}"
  name        = "TrustedIPRanges"
}
```

## Step 3: Configure Findings via EventBridge

```hcl
# Route HIGH severity GuardDuty findings to SNS

resource "aws_cloudwatch_event_rule" "guardduty_high_findings" {
  name        = "${var.project_name}-guardduty-high-severity"
  description = "Alert on HIGH severity GuardDuty findings"

  event_pattern = jsonencode({
    source        = ["aws.guardduty"]
    "detail-type" = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7] }]  # HIGH (7-8.9) and CRITICAL (9-10)
    }
  })
}

resource "aws_cloudwatch_event_target" "guardduty_sns" {
  rule      = aws_cloudwatch_event_rule.guardduty_high_findings.name
  target_id = "guardduty-alert"
  arn       = var.security_sns_topic_arn

  input_transformer {
    input_paths = {
      severity    = "$.detail.severity"
      type        = "$.detail.type"
      description = "$.detail.description"
      account     = "$.account"
      region      = "$.region"
    }
    input_template = "\"GuardDuty ALERT: <type> in account <account> (<region>). Severity: <severity>. <description>\""
  }
}
```

## Step 4: Multi-Account Organization Setup

```hcl
# Designate an admin account for Organization-wide GuardDuty
resource "aws_guardduty_organization_admin_account" "main" {
  admin_account_id = var.security_account_id
}

# In the admin account: configure organization settings
resource "aws_guardduty_organization_configuration" "main" {
  auto_enable_organization_members = "ALL"  # or "NEW"
  detector_id                      = aws_guardduty_detector.main.id

  datasources {
    s3_logs {
      auto_enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          auto_enable = true
        }
      }
    }
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check GuardDuty findings
aws guardduty list-findings \
  --detector-id <detector-id> \
  --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}'
```

## Conclusion

GuardDuty should be enabled in every AWS account and region-the cost is based on data analyzed and is minimal compared to the security value. Enable S3 protection and EKS audit log monitoring for comprehensive coverage beyond EC2 and CloudTrail. For multi-account deployments, use Organizations integration to auto-enable GuardDuty in all current and future accounts and centralize findings in a security account.
