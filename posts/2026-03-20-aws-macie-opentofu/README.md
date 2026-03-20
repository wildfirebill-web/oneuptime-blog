# How to Configure AWS Macie with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Macie, Data Security, S3, PII Detection, Infrastructure as Code

Description: Learn how to enable and configure AWS Macie with OpenTofu to automatically discover and protect sensitive data like PII and credentials in S3 buckets.

## Introduction

AWS Macie uses machine learning to automatically discover, classify, and protect sensitive data in S3. It identifies personally identifiable information (PII), financial data, credentials, and custom sensitive data patterns. Macie also provides a security posture assessment of S3 buckets—flagging publicly accessible buckets, unencrypted buckets, and buckets shared with external accounts.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with Macie and S3 permissions
- S3 buckets to protect

## Step 1: Enable Macie

```hcl
resource "aws_macie2_account" "main" {
  finding_publishing_frequency = "SIX_HOURS"  # FIFTEEN_MINUTES, ONE_HOUR, or SIX_HOURS
  status                       = "ENABLED"
}
```

## Step 2: Create Classification Jobs

```hcl
# One-time classification job for sensitive data discovery
resource "aws_macie2_classification_job" "full_scan" {
  depends_on = [aws_macie2_account.main]

  name       = "${var.project_name}-full-scan"
  job_type   = "ONE_TIME"  # or SCHEDULED

  s3_job_definition {
    bucket_definitions {
      account_id = data.aws_caller_identity.current.account_id
      buckets    = var.s3_bucket_names  # List of buckets to scan
    }

    # Scope to specific prefixes
    scoping {
      includes {
        and {
          simple_scope_term {
            comparator = "STARTS_WITH"
            key        = "PREFIX"
            values     = ["data/", "uploads/", "user-files/"]
          }
        }
      }
    }
  }

  tags = {
    Name = "${var.project_name}-full-scan"
  }
}

# Scheduled weekly classification job
resource "aws_macie2_classification_job" "weekly_scan" {
  depends_on = [aws_macie2_account.main]

  name     = "${var.project_name}-weekly-scan"
  job_type = "SCHEDULED"

  schedule_frequency {
    weekly_schedule {
      day_of_week = "MONDAY"
    }
  }

  s3_job_definition {
    bucket_definitions {
      account_id = data.aws_caller_identity.current.account_id
      buckets    = var.s3_bucket_names
    }
  }

  tags = {
    Name = "${var.project_name}-weekly-scan"
  }
}
```

## Step 3: Create Custom Data Identifier

```hcl
# Custom identifier for your internal account numbers
resource "aws_macie2_custom_data_identifier" "account_number" {
  depends_on = [aws_macie2_account.main]

  name        = "InternalAccountNumber"
  description = "Detect internal customer account numbers"

  regex             = "ACC-[0-9]{8}"  # Matches ACC-12345678
  maximum_match_distance = 50         # Context characters around match

  keywords          = ["account", "customer", "client"]  # Must appear near the pattern
  ignore_words      = ["ACCOUNT_TEMPLATE", "ACC-00000000"]  # Known safe values

  tags = {
    Name = "${var.project_name}-account-number-identifier"
  }
}
```

## Step 4: Route Macie Findings to SNS

```hcl
resource "aws_cloudwatch_event_rule" "macie_high_findings" {
  name        = "${var.project_name}-macie-high-findings"
  description = "Alert on HIGH severity Macie findings"

  event_pattern = jsonencode({
    source        = ["aws.macie"]
    "detail-type" = ["Macie Finding"]
    detail = {
      severity = {
        description = ["High", "Critical"]
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "macie_sns" {
  rule      = aws_cloudwatch_event_rule.macie_high_findings.name
  target_id = "macie-alert"
  arn       = var.security_sns_topic_arn

  input_transformer {
    input_paths = {
      title    = "$.detail.title"
      severity = "$.detail.severity.description"
      bucket   = "$.detail.resourcesAffected.s3Bucket.name"
      account  = "$.account"
    }
    input_template = "\"Macie <severity> Finding: <title>\\nBucket: <bucket>\\nAccount: <account>\""
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# View findings
aws macie2 list-findings \
  --filter-criteria '{"severity":{"gte":7}}'
```

## Conclusion

Macie is essential for organizations handling PII or financial data in S3—it continuously monitors bucket security posture and can discover sensitive data you didn't know existed. Start with a one-time scan of your most sensitive buckets to establish a baseline, then enable scheduled scans for ongoing discovery. Use custom data identifiers to detect business-specific sensitive patterns that Macie's built-in managed identifiers don't cover, and route HIGH/CRITICAL findings to your security team for immediate investigation.
