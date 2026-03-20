# How to Use the -parallelism Flag in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use the -parallelism flag in OpenTofu to control how many resources are created, updated, or destroyed concurrently.

## Introduction

OpenTofu creates, updates, and destroys resources concurrently to reduce deployment time. The `-parallelism` flag controls the maximum number of concurrent operations. The default is 10. Increasing it speeds up large deployments; decreasing it helps avoid API rate limits or resource creation ordering issues.

## Basic Usage

```bash
# Default parallelism (10 concurrent operations)
tofu apply

# Increase concurrency for faster deployments
tofu apply -parallelism=20

# Decrease to avoid rate limits
tofu apply -parallelism=5
```

## Default Behavior

```bash
# OpenTofu runs up to 10 operations simultaneously
tofu apply -parallelism=10  # Same as default

# During a large apply with 50 resources:
# - 10 start immediately
# - As each finishes, another starts
# - Dependency ordering is always respected
```

## When to Increase Parallelism

```bash
# Large configurations with many independent resources
# Example: provisioning 50 EC2 instances
tofu apply -parallelism=50

# Example: creating many S3 buckets or IAM policies
tofu apply -parallelism=30
```

Increasing parallelism reduces total deployment time for configurations with many independent resources.

## When to Decrease Parallelism

```bash
# Provider has aggressive API rate limits
tofu apply -parallelism=3

# AWS API rate limits for certain services (e.g., IAM, Route53)
tofu apply -parallelism=5

# Debugging: run one resource at a time to see ordering issues
tofu apply -parallelism=1
```

## Provider-Specific Rate Limits

Some services have strict rate limits:

```bash
# Route53 changes can trigger rate limiting at high parallelism
tofu apply -parallelism=5

# IAM API has lower limits than EC2
# Reduce parallelism if seeing ThrottlingException errors
tofu apply -parallelism=4
```

## Works with Dependency Graph

Parallelism is always constrained by the dependency graph:

```hcl
# These two resources have no dependency — can run in parallel
resource "aws_s3_bucket" "a" { bucket = "bucket-a" }
resource "aws_s3_bucket" "b" { bucket = "bucket-b" }

# This depends on "a" — runs after a, even with high parallelism
resource "aws_s3_bucket_versioning" "a" {
  bucket = aws_s3_bucket.a.id
}
```

Even with `-parallelism=100`, `aws_s3_bucket_versioning.a` waits for `aws_s3_bucket.a`.

## Destroy with Parallelism

```bash
# Speed up teardown with higher parallelism
tofu destroy -parallelism=30 -auto-approve

# Or slow down to avoid dependency issues during destroy
tofu destroy -parallelism=5 -auto-approve
```

## Setting Default Parallelism

```bash
# Set via environment variable (persistent default)
export TF_CLI_ARGS_plan="-parallelism=20"
export TF_CLI_ARGS_apply="-parallelism=20"
```

## Parallelism and State Refresh

Parallelism also affects the state refresh step:

```bash
# 10 concurrent API calls during refresh by default
tofu plan  # Refreshes up to 10 resources simultaneously

# Reduce if experiencing API errors during refresh
tofu plan -parallelism=5
```

## Debugging with Single Parallelism

```bash
# Run completely sequentially for debugging
tofu apply -parallelism=1

# This reveals exact ordering and makes logs easier to read
```

## Conclusion

The `-parallelism` flag balances deployment speed against API rate limits. The default of 10 works well for most configurations. Increase it (20-50) for large, independent resource sets to reduce deployment time. Decrease it (2-5) when hitting throttling errors from cloud provider APIs, or set it to 1 for sequential debugging. Dependency ordering is always respected regardless of the parallelism setting.
