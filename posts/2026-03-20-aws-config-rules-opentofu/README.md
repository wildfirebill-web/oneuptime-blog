# How to Set Up AWS Config Rules with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Configs, Compliance, Infrastructure Governance, Infrastructure as Code

Description: Learn how to configure AWS Config Rules with OpenTofu to continuously evaluate AWS resource configurations for compliance with security and operational best practices.

## Introduction

AWS Config continuously records resource configurations and evaluates them against rules. Config Rules can use AWS-managed rules (over 200 pre-built) or custom Lambda-backed rules to check configurations like S3 bucket encryption, security group rules, and IAM password policies. Non-compliant resources are flagged for remediation.

## Prerequisites

- OpenTofu v1.6+
- AWS Config recorder enabled
- AWS credentials with Config permissions

## Step 1: Enable AWS Config Recorder

```hcl
resource "aws_config_configuration_recorder" "main" {
  name     = "${var.project_name}-config-recorder"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported                 = true   # Record all supported resources
    include_global_resource_types = true   # Include IAM, Route 53, etc.
  }
}

resource "aws_iam_role" "config" {
  name = "${var.project_name}-config-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "config.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "config" {
  role       = aws_iam_role.config.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWS_ConfigRole"
}

# S3 bucket for Config delivery

resource "aws_s3_bucket" "config" {
  bucket = "${var.project_name}-config-${data.aws_caller_identity.current.account_id}"
}

resource "aws_config_delivery_channel" "main" {
  name           = "${var.project_name}-config-delivery"
  s3_bucket_name = aws_s3_bucket.config.id

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true

  depends_on = [aws_config_delivery_channel.main]
}
```

## Step 2: Enable AWS Managed Config Rules

```hcl
# S3 buckets must not be publicly accessible
resource "aws_config_config_rule" "s3_public_access" {
  name        = "s3-bucket-public-read-prohibited"
  description = "S3 buckets must block public read access"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

# Root account MFA must be enabled
resource "aws_config_config_rule" "root_mfa" {
  name        = "root-account-mfa-enabled"
  description = "Root account must have MFA enabled"

  source {
    owner             = "AWS"
    source_identifier = "ROOT_ACCOUNT_MFA_ENABLED"
  }

  # This rule applies at the account level
  depends_on = [aws_config_configuration_recorder.main]
}

# Ensure EBS volumes are encrypted
resource "aws_config_config_rule" "ebs_encryption" {
  name        = "ec2-ebs-encryption-by-default"
  description = "EC2 EBS volumes must be encrypted by default"

  source {
    owner             = "AWS"
    source_identifier = "EC2_EBS_ENCRYPTION_BY_DEFAULT"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

# CloudTrail must be enabled
resource "aws_config_config_rule" "cloudtrail_enabled" {
  name        = "cloud-trail-enabled"
  description = "CloudTrail must be enabled with log file validation"

  source {
    owner             = "AWS"
    source_identifier = "CLOUD_TRAIL_ENABLED"
  }

  input_parameters = jsonencode({
    s3BucketName = var.cloudtrail_bucket_name
  })

  depends_on = [aws_config_configuration_recorder.main]
}
```

## Step 3: Enable Conformance Pack

```hcl
# Deploy CIS AWS Benchmark conformance pack
resource "aws_config_conformance_pack" "cis" {
  name = "${var.project_name}-cis-benchmark"

  template_s3_uri = "s3://aws-configservice-us-east-1/conformance-packs/Operational-Best-Practices-for-CIS-Critical-Security-Controls-v8-IG1.yaml"

  depends_on = [aws_config_configuration_recorder.main]
}
```

## Step 4: Configure Automatic Remediation

```hcl
# Auto-remediate: Enable encryption on non-compliant S3 buckets
resource "aws_config_remediation_configuration" "s3_encryption" {
  config_rule_name = aws_config_config_rule.s3_public_access.name

  resource_type  = "AWS::S3::Bucket"
  target_type    = "SSM_DOCUMENT"
  target_id      = "AWS-EnableS3BucketEncryption"
  automatic      = false  # Set to true for automatic remediation

  parameter {
    name           = "BucketName"
    resource_value = "RESOURCE_ID"
  }

  parameter {
    name         = "SSEAlgorithm"
    static_value = "AES256"
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Check compliance summary
aws config get-compliance-summary-by-config-rule \
  --config-rule-names s3-bucket-public-read-prohibited
```

## Conclusion

AWS Config Rules provide continuous compliance monitoring that catches drift between infrastructure changes and your security baseline. Start with the CIS AWS Benchmark conformance pack for comprehensive coverage across the most impactful security controls. Enable automatic remediation selectively for low-risk, well-understood fixes (like enabling S3 encryption), but require manual approval for remediations that could impact availability, like modifying security group rules.
