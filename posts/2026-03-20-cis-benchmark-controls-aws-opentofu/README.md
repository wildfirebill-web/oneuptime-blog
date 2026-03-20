# How to Implement CIS Benchmark Controls with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, CIS Benchmark, AWS Security, Compliance, Infrastructure as Code

Description: Learn how to implement CIS AWS Foundations Benchmark controls with OpenTofu to establish a security baseline for your AWS accounts.

The CIS AWS Foundations Benchmark provides prescriptive guidance for securing AWS accounts. OpenTofu lets you codify these controls as resources, making compliance verifiable and reproducible.

## CIS Control Categories

| Section | Controls |
|---|---|
| 1. IAM | Password policy, MFA, access key rotation |
| 2. Storage | S3 public access block, CloudTrail S3 logging |
| 3. Logging | CloudTrail, VPC Flow Logs, Config |
| 4. Monitoring | CloudWatch alarms for unauthorized activity |
| 5. Networking | VPC defaults, security group restrictions |

## Section 1: IAM Controls

```hcl
# CIS 1.8 — Ensure IAM password policy requires minimum 14-character passwords
resource "aws_iam_account_password_policy" "cis" {
  minimum_password_length        = 14
  require_lowercase_characters   = true
  require_numbers                = true
  require_uppercase_characters   = true
  require_symbols                = true
  allow_users_to_change_password = true
  max_password_age               = 90
  password_reuse_prevention      = 24
  hard_expiry                    = false
}
```

## Section 2: Storage Controls

```hcl
# CIS 2.1.1 — Ensure S3 bucket public access is blocked at account level
resource "aws_s3_account_public_access_block" "cis" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## Section 3: Logging Controls

```hcl
# CIS 3.1 — Ensure CloudTrail is enabled in all regions
resource "aws_cloudtrail" "cis" {
  name                          = "cis-cloudtrail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
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

# CIS 3.2 — Ensure CloudTrail log file validation is enabled (set above)
# CIS 3.7 — Ensure CloudTrail logs are encrypted at rest
resource "aws_cloudtrail" "encrypted" {
  kms_key_id = aws_kms_key.cloudtrail.arn
  # ... other config
}
```

## Section 4: Monitoring Controls

```hcl
# CIS 4.1 — Unauthorized API calls alarm
resource "aws_cloudwatch_metric_alarm" "unauthorized_api" {
  alarm_name          = "cis-unauthorized-api-calls"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "UnauthorizedAttemptCount"
  namespace           = "CISBenchmark"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}

# CIS 4.3 — Root account usage alarm
resource "aws_cloudwatch_metric_alarm" "root_usage" {
  alarm_name          = "cis-root-account-usage"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 1
  metric_name         = "RootAccountUsage"
  namespace           = "CISBenchmark"
  period              = 300
  statistic           = "Sum"
  threshold           = 1
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

## Section 5: Networking Controls

```hcl
# CIS 5.4 — Ensure the default security group restricts all traffic
resource "aws_default_security_group" "cis" {
  vpc_id = aws_vpc.main.id
  # No ingress or egress rules = deny all
}

# CIS 5.1 — Ensure no security groups allow unrestricted SSH access
# (Validated via AWS Config rule or custom policy check)
```

## Using a CIS Benchmark Module

```hcl
module "cis_baseline" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "~> 5.0"
  # or use a dedicated CIS module
}
```

## Conclusion

Implementing CIS Benchmark controls with OpenTofu makes compliance declarative and auditable. Codify IAM password policies, S3 public access blocks, CloudTrail configuration, CloudWatch alarms, and VPC security group defaults as Terraform resources. Run `tofu plan` in CI to detect drift from the security baseline before it reaches production.
