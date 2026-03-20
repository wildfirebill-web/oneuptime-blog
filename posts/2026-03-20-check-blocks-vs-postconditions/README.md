# How to Use Check Blocks vs Postconditions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Testing

Description: Understand the differences between check blocks and postconditions in OpenTofu and know when to use each for infrastructure validation.

## Introduction

OpenTofu provides two validation mechanisms that run after resource creation: check blocks and postconditions. While they may seem similar, they behave differently and serve distinct purposes. Choosing between them depends on whether you want warnings or blocking errors, and whether you need access to the resource's `self` object.

## Key Differences

| Feature | `check` block | `postcondition` |
|---------|--------------|-----------------|
| Lives in | Top-level configuration | Resource `lifecycle` block |
| Runs | Every plan and apply | After resource create/update/read |
| On failure | Warning (non-blocking) | Error (taints resource) |
| `self` access | No | Yes |
| Scoped data source | Yes (one per check) | No |
| Timing | After all resources applied | During resource operation |

## Postcondition Example

Postconditions are defined inside a resource and can reference `self`:

```hcl
resource "aws_db_instance" "main" {
  identifier        = "prod-db"
  engine            = "postgres"
  instance_class    = var.instance_class
  multi_az          = true
  deletion_protection = true

  lifecycle {
    postcondition {
      # self references the resource after creation
      condition     = self.multi_az == true
      error_message = "Database must be Multi-AZ. Provider returned: ${self.multi_az}"
    }
    postcondition {
      condition     = self.deletion_protection == true
      error_message = "Deletion protection must be enabled"
    }
  }
}
```

## Check Block Example

Check blocks are top-level and can use scoped data sources for live queries:

```hcl
check "database_is_available" {
  data "aws_db_instance" "check_db" {
    db_instance_identifier = aws_db_instance.main.identifier
  }

  assert {
    # Check blocks do NOT have self - use data source or direct resource reference
    condition     = data.aws_db_instance.check_db.db_instance_status == "available"
    error_message = "Database is not in available state: ${data.aws_db_instance.check_db.db_instance_status}"
  }
}
```

## When to Use Postconditions

Use postconditions when:

1. You want to block the apply if the resource is created incorrectly
2. You need to validate the actual resource attributes via `self`
3. You want to catch cases where the provider ignores your configuration

```hcl
resource "aws_s3_bucket" "state" {
  bucket = "my-state-bucket"

  lifecycle {
    postcondition {
      # Verify the bucket was created with the exact name we requested
      condition     = self.bucket == "my-state-bucket"
      error_message = "Bucket created with unexpected name: ${self.bucket}"
    }
  }
}
```

## When to Use Check Blocks

Use check blocks when:

1. You want non-blocking warnings (gradual adoption of rules)
2. You need to query external state via a scoped data source
3. You want continuous validation on every plan (not just when resource changes)
4. You're doing health checks or compliance validation

```hcl
check "website_up" {
  data "http" "health" {
    url = "https://${aws_lb.main.dns_name}/health"
  }

  assert {
    condition     = data.http.health.status_code == 200
    error_message = "Health check failed: ${data.http.health.status_code}"
  }
}
```

## Using Both Together

In practice, use postconditions for correctness guarantees and check blocks for ongoing compliance:

```hcl
# Postcondition: Block if encryption isn't actually enabled

resource "aws_ebs_volume" "data" {
  size      = 100
  encrypted = true

  lifecycle {
    postcondition {
      condition     = self.encrypted == true
      error_message = "Volume was created without encryption"
    }
  }
}

# Check block: Ongoing compliance check querying live state
check "volume_encryption_compliance" {
  assert {
    condition     = aws_ebs_volume.data.encrypted == true
    error_message = "EBS volume encryption compliance check failed"
  }
}
```

## Failure Output Comparison

Postcondition failure (blocks apply, taints resource):
```hcl
Error: Resource postcondition failed
  on main.tf line 12, in resource "aws_ebs_volume" "data":
Volume was created without encryption
Because of this error, OpenTofu has tainted the resource.
```

Check block failure (warning only, apply succeeds):
```text
Warning: Check block assertion failed
  on main.tf line 22, in check "volume_encryption_compliance":
EBS volume encryption compliance check failed
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

## Conclusion

Use postconditions when you need hard guarantees about resource state and want to block incorrect deployments. Use check blocks for continuous compliance monitoring, health checks, and non-blocking validation that won't halt your pipeline. The two mechanisms complement each other: postconditions enforce correctness at creation time, while check blocks provide ongoing visibility into infrastructure state.
