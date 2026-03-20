# How to Write Assertions in Check Blocks in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Validation

Description: Learn how to write effective assertions in OpenTofu check blocks, including conditions, error messages, and patterns for common infrastructure validation scenarios.

## Introduction

The `assert` block inside a `check` block defines a condition that should be true and an error message displayed when it is not. Writing clear, specific assertions with helpful error messages makes check blocks valuable as operational guardrails that guide operators toward the right fix.

## Basic Assert Syntax

```hcl
check "example" {
  assert {
    condition     = <boolean expression>
    error_message = "Human-readable description of what went wrong."
  }
}
```

## String Assertions

```hcl
check "naming_conventions" {
  # Assert bucket name starts with company prefix
  assert {
    condition     = startswith(aws_s3_bucket.data.bucket, "acme-")
    error_message = "Bucket name '${aws_s3_bucket.data.bucket}' must start with 'acme-'."
  }

  # Assert no uppercase in bucket name
  assert {
    condition     = aws_s3_bucket.data.bucket == lower(aws_s3_bucket.data.bucket)
    error_message = "Bucket name must be lowercase. Got: '${aws_s3_bucket.data.bucket}'"
  }
}
```

## Numeric Assertions

```hcl
check "capacity_requirements" {
  assert {
    condition     = aws_autoscaling_group.app.min_size >= 2
    error_message = "Minimum instance count must be at least 2 for HA. Got: ${aws_autoscaling_group.app.min_size}"
  }

  assert {
    condition     = aws_autoscaling_group.app.max_size <= 100
    error_message = "Maximum instances must not exceed 100 to control costs. Got: ${aws_autoscaling_group.app.max_size}"
  }
}
```

## Collection Assertions

```hcl
check "tagging_policy" {
  assert {
    condition     = length(aws_instance.web.tags) >= 3
    error_message = "Resources must have at least 3 tags. Found: ${length(aws_instance.web.tags)}"
  }

  assert {
    condition     = contains(keys(aws_instance.web.tags), "CostCenter")
    error_message = "Resources must have a 'CostCenter' tag for cost allocation."
  }
}
```

## Boolean and Null Assertions

```hcl
check "security_settings" {
  assert {
    condition     = aws_s3_bucket_public_access_block.data.block_public_acls == true
    error_message = "S3 bucket must block public ACLs."
  }

  assert {
    condition     = aws_db_instance.main.deletion_protection == true
    error_message = "RDS deletion protection must be enabled in production."
  }

  assert {
    condition     = aws_db_instance.main.final_snapshot_identifier != null
    error_message = "RDS must have a final snapshot identifier set."
  }
}
```

## Regex Assertions

```hcl
check "format_validation" {
  assert {
    condition     = can(regex("^[a-z0-9-]+$", aws_s3_bucket.data.bucket))
    error_message = "Bucket name '${aws_s3_bucket.data.bucket}' contains invalid characters. Use only lowercase letters, numbers, and hyphens."
  }
}
```

## CIDR Assertions

```hcl
check "network_configuration" {
  assert {
    condition     = cidrcontains("10.0.0.0/8", aws_vpc.main.cidr_block)
    error_message = "VPC CIDR '${aws_vpc.main.cidr_block}' must be within the 10.0.0.0/8 private address space."
  }
}
```

## Writing Helpful Error Messages

Good error messages explain:
1. What the condition checks
2. What the actual value is
3. What the expected value or pattern is

```hcl
# Poor error message

assert {
  condition     = aws_instance.web.instance_type == "t3.large"
  error_message = "Wrong instance type."
}

# Good error message
assert {
  condition     = aws_instance.web.instance_type == "t3.large"
  error_message = "Production web servers must use t3.large. Current type: ${aws_instance.web.instance_type}. Update var.instance_type."
}
```

## Multiple Assertions per Check Block

```hcl
check "rds_production_requirements" {
  assert {
    condition     = aws_db_instance.main.multi_az == true
    error_message = "Production RDS must be Multi-AZ for high availability."
  }

  assert {
    condition     = aws_db_instance.main.backup_retention_period >= 7
    error_message = "RDS backup retention must be at least 7 days. Current: ${aws_db_instance.main.backup_retention_period}"
  }

  assert {
    condition     = aws_db_instance.main.storage_encrypted == true
    error_message = "RDS storage encryption must be enabled."
  }

  assert {
    condition     = aws_db_instance.main.deletion_protection == true
    error_message = "RDS deletion protection must be enabled."
  }
}
```

## Using try() for Safe Assertions

```hcl
check "optional_feature_check" {
  assert {
    # Use try() when the attribute might not exist
    condition     = try(aws_s3_bucket.data.versioning[0].enabled, false) == true
    error_message = "S3 bucket versioning should be enabled for state storage."
  }
}
```

## Conclusion

Effective check block assertions use specific conditions that directly test the invariant you care about, and error messages that tell operators exactly what is wrong and how to fix it. Include the actual value in the error message using string interpolation. Group related assertions in a single check block with a descriptive name. Use `try()` for optional attributes that may not always exist. Clear assertions and error messages make check blocks a self-documenting compliance policy for your infrastructure.
