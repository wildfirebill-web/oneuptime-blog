# How to Handle Concurrent State Access in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, State Locking, DynamoDB, Concurrency, CI/CD

Description: Learn how to configure and troubleshoot state locking in OpenTofu to prevent concurrent state access conflicts when multiple engineers or CI jobs run applies simultaneously.

## Introduction

OpenTofu uses state locking to prevent concurrent modifications. Without locking, two simultaneous `apply` operations could corrupt the state file. Understanding locking configuration and troubleshooting stale locks is essential for team environments.

## Configuring State Locking with DynamoDB

```hcl
# S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-company-tofu-state"
    key            = "prod/app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-state-lock"  # DynamoDB table name
  }
}
```

```hcl
# Create the DynamoDB lock table
resource "aws_dynamodb_table" "state_lock" {
  name         = "tofu-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  server_side_encryption {
    enabled = true
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Purpose = "OpenTofu state locking"
  }
}
```

## Checking and Removing Stale Locks

When a CI job crashes mid-apply, the lock may remain. Check and remove stale locks:

```bash
# Check if a state is currently locked
aws dynamodb get-item \
  --table-name tofu-state-lock \
  --key '{"LockID": {"S": "my-company-tofu-state/prod/app/terraform.tfstate"}}' \
  --output json

# If the lock is stale (process is no longer running), force-unlock
# Get the lock ID from the error message when trying to run tofu
tofu force-unlock <LOCK_ID>

# Example:
# Error: Error acquiring the state lock: ConditionalCheckFailedException
# Lock Info:
#   ID:        abc123-def456-ghi789
#   Path:      my-company-tofu-state/prod/app/terraform.tfstate
#   Created:   2026-03-20 10:15:00
#   Info:      {"ID":"abc123","Operation":"apply","Who":"ci-runner"}
#
tofu force-unlock abc123-def456-ghi789
```

## Lock Timeout Configuration

```hcl
# Override the default lock wait timeout
# -lock-timeout=0s means fail immediately if locked
# -lock-timeout=5m means wait up to 5 minutes

resource "null_resource" "example" {}
```

```bash
# In CI/CD: fail fast if locked (another job is running)
tofu apply -lock-timeout=0s -auto-approve

# For interactive use: wait up to 10 minutes
tofu apply -lock-timeout=10m
```

## CI/CD Concurrent Lock Management

In CI systems with parallel jobs, use workspace-level locks and queue operations:

```yaml
# GitHub Actions: prevent concurrent applies per environment
concurrency:
  group: tofu-${{ github.ref }}-${{ matrix.environment }}
  cancel-in-progress: false  # Queue, don't cancel
```

## Per-Workspace Lock Tables

For large teams, use separate lock tables per environment to avoid contention:

```hcl
locals {
  lock_tables = {
    dev     = "tofu-lock-dev"
    staging = "tofu-lock-staging"
    prod    = "tofu-lock-prod"
  }
}

resource "aws_dynamodb_table" "lock_tables" {
  for_each = local.lock_tables

  name         = each.value
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Monitoring Lock Table Activity

```hcl
# CloudWatch alarm for excessive lock wait times
resource "aws_cloudwatch_metric_alarm" "lock_contention" {
  alarm_name          = "tofu-lock-contention"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ConsumedWriteCapacityUnits"
  namespace           = "AWS/DynamoDB"
  period              = 60
  statistic           = "Sum"
  threshold           = 100  # Adjust based on team size
  dimensions = {
    TableName = aws_dynamodb_table.state_lock.name
  }
}
```

## Conclusion

DynamoDB state locking is the standard mechanism for coordinating concurrent OpenTofu access. Always configure the DynamoDB table alongside your S3 backend — running without locking in a team environment will eventually cause state corruption. When removing stale locks with `force-unlock`, first confirm the process that held the lock is no longer running.
