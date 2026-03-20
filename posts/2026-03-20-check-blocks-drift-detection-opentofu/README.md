# How to Use Check Blocks for Drift Detection in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Check Blocks, Drift Detection, Validation, Infrastructure as Code, Testing

Description: Learn how to use OpenTofu's check blocks to define post-deployment assertions that detect configuration drift and verify infrastructure health during every plan and apply.

## Introduction

OpenTofu's `check` block (introduced in 1.5) lets you define custom assertions about your infrastructure that are evaluated during every plan and apply. Unlike `validation` blocks (which check input variables), `check` blocks verify the actual state of deployed resources — making them a powerful drift detection tool.

## Basic Check Block Syntax

```hcl
# A check block defines an assertion about deployed infrastructure
check "web_instance_running" {
  # Optional: scope the check to specific data
  data "aws_instance" "web_check" {
    instance_id = aws_instance.web.id
  }

  # The assertion to verify
  assert {
    condition     = data.aws_instance.web_check.instance_state == "running"
    error_message = "Web instance ${aws_instance.web.id} is not in running state."
  }
}
```

## Checking Security Group Rules

```hcl
# Detect if someone removed a critical security group rule
check "https_ingress_present" {
  data "aws_security_group" "web_check" {
    id = aws_security_group.web.id
  }

  assert {
    condition = anytrue([
      for rule in data.aws_security_group.web_check.ingress : rule.from_port == 443
    ])
    error_message = "DRIFT DETECTED: HTTPS ingress rule missing from web security group. Someone may have removed it manually."
  }
}
```

## Checking S3 Bucket Encryption

```hcl
check "s3_encryption_enabled" {
  data "aws_s3_bucket" "app_check" {
    bucket = aws_s3_bucket.app.id
  }

  assert {
    condition     = data.aws_s3_bucket.app_check.bucket != ""
    error_message = "S3 bucket check failed — bucket may have been deleted externally."
  }
}

# More specific: check server-side encryption
check "s3_sse_active" {
  data "aws_s3_bucket_server_side_encryption_configuration" "app" {
    bucket = aws_s3_bucket.app.id
  }

  assert {
    condition = length(data.aws_s3_bucket_server_side_encryption_configuration.app.rule) > 0
    error_message = "DRIFT: S3 bucket encryption was disabled externally."
  }
}
```

## Checking RDS Instance Status

```hcl
check "rds_instance_available" {
  data "aws_db_instance" "main_check" {
    db_instance_identifier = aws_db_instance.main.identifier
  }

  assert {
    condition     = data.aws_db_instance.main_check.db_instance_status == "available"
    error_message = "RDS instance ${aws_db_instance.main.identifier} is not in available state — current: ${data.aws_db_instance.main_check.db_instance_status}"
  }
}
```

## Checking DNS Resolution

```hcl
# Verify a DNS record resolves correctly after changes
check "api_dns_resolves" {
  data "dns_a_record_set" "api_check" {
    host = "api.example.com"
  }

  assert {
    condition     = length(data.dns_a_record_set.api_check.addrs) > 0
    error_message = "DRIFT: api.example.com DNS record is not resolving. Check Route53 configuration."
  }
}
```

## Check Block Behavior

- **Warnings only**: Failed assertions produce warnings, not errors. Apply still succeeds.
- **Plan time**: Checks run during `tofu plan` and `tofu apply`
- **Scope**: Each check block can include one `data` source and multiple `assert` blocks

## Converting Warning to Error (using -compact-warnings)

```bash
# Check warnings are visible but not blocking by default
tofu plan 2>&1 | grep -A 3 "Warning:"

# To treat check warnings as errors in CI:
tofu plan 2>&1 | tee plan.log
if grep -q "check block assertions failed" plan.log; then
  echo "ERROR: Check assertions failed — drift detected!"
  exit 1
fi
```

## Conclusion

Check blocks provide a declarative, co-located way to verify the health and configuration of deployed resources on every plan. They bridge the gap between apply-time validation and continuous monitoring, catching drift in the same run that checks for configuration changes. Use them to assert critical security controls (encryption enabled, ports not open) and operational requirements (instances running, DNS resolving) that could be changed out-of-band.
