# How to Use Check Blocks for Infrastructure Validation in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Testing

Description: Learn how to use OpenTofu check blocks to continuously validate infrastructure assumptions and detect configuration drift with custom assertions.

## Introduction

Check blocks (introduced in OpenTofu 1.5) allow you to write post-deployment assertions about your infrastructure. Unlike preconditions and postconditions (which run during resource operations), check blocks run as a separate step during every plan and apply, making them suitable for ongoing infrastructure validation.

## Basic Check Block Syntax

```hcl
check "website_is_live" {
  assert {
    condition     = can(regex("^2", data.http.website.status_code))
    error_message = "Website returned a non-2xx status code: ${data.http.website.status_code}"
  }
}
```

## Check Block with Data Source

Check blocks can include an optional scoped data source:

```hcl
check "s3_bucket_exists" {
  # Scoped data source — only available within this check block
  data "aws_s3_bucket" "state" {
    bucket = "my-terraform-state"
  }

  assert {
    condition     = data.aws_s3_bucket.state.bucket == "my-terraform-state"
    error_message = "Expected state bucket not found"
  }
}
```

## Multiple Assertions in One Check Block

```hcl
check "database_configuration" {
  data "aws_db_instance" "main" {
    db_instance_identifier = aws_db_instance.main.identifier
  }

  assert {
    condition     = data.aws_db_instance.main.multi_az == true
    error_message = "Production database must have Multi-AZ enabled"
  }

  assert {
    condition     = data.aws_db_instance.main.deletion_protection == true
    error_message = "Production database must have deletion protection enabled"
  }

  assert {
    condition     = data.aws_db_instance.main.backup_retention_period >= 7
    error_message = "Database backup retention must be at least 7 days, got: ${data.aws_db_instance.main.backup_retention_period}"
  }
}
```

## Check Block Behavior

Check blocks produce **warnings**, not errors:

```bash
tofu apply

# Output:
# ╷
# │ Warning: Check block assertion failed
# │
# │   on main.tf line 42, in check "database_configuration":
# │   42:     condition = data.aws_db_instance.main.multi_az == true
# │
# │ Production database must have Multi-AZ enabled
# ╵
# Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
# (apply still succeeds — checks are warnings, not blockers)
```

## Using Check Blocks for Security Compliance

```hcl
check "s3_encryption_enabled" {
  data "aws_s3_bucket" "app" {
    bucket = aws_s3_bucket.app.id
  }

  assert {
    condition     = length(data.aws_s3_bucket.app.server_side_encryption_configuration) > 0
    error_message = "S3 bucket must have server-side encryption enabled"
  }
}

check "no_public_s3_access" {
  assert {
    condition     = aws_s3_bucket_public_access_block.app.block_public_acls == true
    error_message = "S3 bucket must have public ACL blocking enabled"
  }
}
```

## Tagging Compliance Checks

```hcl
check "required_tags_present" {
  assert {
    condition = can(aws_instance.web.tags["Environment"]) && can(aws_instance.web.tags["Owner"])
    error_message = "EC2 instance missing required tags: Environment and Owner"
  }
}

check "tag_values_valid" {
  assert {
    condition     = contains(["dev", "staging", "prod"], aws_instance.web.tags["Environment"])
    error_message = "Environment tag must be one of: dev, staging, prod"
  }
}
```

## Check vs Precondition/Postcondition

| Feature | check block | precondition | postcondition |
|---------|-------------|-------------|---------------|
| Runs during | Every plan/apply | Resource create/update | After resource create/update |
| On failure | Warning (non-blocking) | Error (blocks operation) | Error (marks resource tainted) |
| Data source support | Yes (scoped) | No | No |

## Conclusion

Check blocks provide continuous infrastructure validation that catches configuration drift and compliance violations during every plan and apply. They're non-blocking (warnings, not errors), making them suitable for gradual adoption of validation rules. Use them for security compliance, tagging policies, and validating dependencies between infrastructure components.
