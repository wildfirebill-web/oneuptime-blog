# How to Implement FedRAMP Controls with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, FedRAMP, Compliance, Government Cloud, Infrastructure as Code

Description: Learn how to implement key FedRAMP technical controls with OpenTofu for cloud systems processing federal government data.

FedRAMP (Federal Risk and Authorization Management Program) is the US government's cloud security authorization framework based on NIST 800-53. It has Moderate and High impact baselines. This guide covers the most infrastructure-relevant controls codifiable with OpenTofu.

## FedRAMP-Specific Requirements

FedRAMP adds specific requirements on top of NIST 800-53:
- **FIPS 140-2** validated cryptographic modules
- **US regions only** (GovCloud required for some agencies)
- **Enhanced logging** with tamper-proof storage
- **Continuous monitoring** with automated evidence collection

## Using AWS GovCloud

```hcl
# FedRAMP workloads typically run in AWS GovCloud

provider "aws" {
  region = "us-gov-west-1"  # or us-gov-east-1

  # GovCloud requires separate credentials
  access_key = var.govcloud_access_key
  secret_key = var.govcloud_secret_key
}
```

## AC-2: Account Management

```hcl
# Automated account management - no manual IAM user creation
resource "aws_organizations_policy" "no_iam_users" {
  name = "FedRAMP-NoDirectIAMUsers"
  type = "SERVICE_CONTROL_POLICY"

  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "DenyIAMUserCreation"
      Effect = "Deny"
      Action = ["iam:CreateUser"]
      Resource = "*"
      # Exception for break-glass accounts
      Condition = {
        StringNotEquals = { "aws:PrincipalTag/Role" = "SecurityAdmin" }
      }
    }]
  })
}
```

## AU-2, AU-3: Comprehensive Audit Logging

```hcl
# FedRAMP requires all S3 data access events to be logged
resource "aws_cloudtrail" "fedramp" {
  name                          = "fedramp-cloudtrail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  kms_key_id                    = aws_kms_key.fedramp.arn  # FIPS-compliant key

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    # Log ALL S3 object operations
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]
    }

    # Log ALL Lambda invocations
    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]
    }
  }
}
```

## SC-8: Transmission Confidentiality (FIPS)

```hcl
# Use FIPS-compliant TLS policy on load balancers
resource "aws_alb_listener" "fedramp_https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"

  # FIPS 140-2 compliant cipher policy
  ssl_policy      = "ELBSecurityPolicy-TLS13-1-2-FIPS-2023-04"
  certificate_arn = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

## IR-6: Incident Reporting - Security Hub

```hcl
resource "aws_securityhub_account" "fedramp" {}

resource "aws_securityhub_standards_subscription" "fedramp" {
  standards_arn = "arn:aws:securityhub:us-gov-west-1::standards/aws-foundational-security-best-practices/v/1.0.0"
  depends_on    = [aws_securityhub_account.fedramp]
}

# FedRAMP-specific findings forwarded to ISSO
resource "aws_securityhub_action_target" "forward_to_isso" {
  name        = "Forward to ISSO"
  identifier  = "ForwardToISSO"
  description = "Forward critical findings to the ISSO"
}
```

## CM-2: Baseline Configuration

```hcl
# AWS Config to detect drift from approved configuration
resource "aws_config_config_rule" "approved_amis" {
  name = "fedramp-approved-amis"

  source {
    owner             = "AWS"
    source_identifier = "APPROVED_AMIS_BY_ID"
  }

  input_parameters = jsonencode({
    amiIds = join(",", var.approved_fedramp_ami_ids)
  })
}
```

## Continuous Monitoring Evidence

```hcl
# Export compliance evidence to an immutable S3 bucket
resource "aws_s3_bucket" "compliance_evidence" {
  bucket = "fedramp-compliance-evidence-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_object_lock_configuration" "evidence" {
  bucket = aws_s3_bucket.compliance_evidence.id

  rule {
    default_retention {
      mode  = "COMPLIANCE"  # Cannot be deleted even by root
      years = 3
    }
  }
}
```

## Conclusion

FedRAMP controls extend NIST 800-53 with government-specific requirements: use AWS GovCloud for data residency, FIPS-compliant cipher policies for TLS, comprehensive event logging for all S3 and Lambda operations, immutable evidence storage, and continuous monitoring via Security Hub. OpenTofu makes these controls verifiable and audit-ready by keeping them in version-controlled configurations.
