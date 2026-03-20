# How to Migrate Infrastructure from CDK for Terraform to OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, CDKTF, Migration, Infrastructure as Code, State Management

Description: Learn how to migrate infrastructure managed by CDK for Terraform (CDKTF) into OpenTofu HCL without recreating cloud resources.

## Introduction

CDK for Terraform (CDKTF) lets you define infrastructure using TypeScript, Python, or other languages that compile to Terraform JSON configuration. Migrating to native OpenTofu HCL simplifies your toolchain by removing the synthesis step, making configurations more readable and directly understandable by operators without programming language knowledge.

## Understanding CDKTF Synthesis

CDKTF compiles your code to JSON Terraform configurations before running.

```bash
# CDKTF synthesizes JSON configurations
cdktf synth

# This creates synthesized JSON in cdktf.out/
ls cdktf.out/stacks/

# The synthesized JSON is valid Terraform/OpenTofu config
cat cdktf.out/stacks/my-stack/cdk.tf.json
```

## Phase 1: Examine the Synthesized JSON

Use CDKTF's synthesized output as a reference for writing HCL.

```bash
# Synthesize and inspect
cdktf synth
cat cdktf.out/stacks/my-stack/cdk.tf.json | jq '.resource'

# Example synthesized resource
# {
#   "aws_s3_bucket": {
#     "MyBucket": {
#       "bucket": "my-app-bucket",
#       "tags": {"Environment": "prod"}
#     }
#   }
# }
```

## Phase 2: Export Current State

CDKTF uses standard Terraform state — you can read it directly.

```bash
# List current state (from within the CDKTF stack directory)
cd cdktf.out/stacks/my-stack
terraform state list   # or tofu state list

# Show a specific resource
tofu state show aws_s3_bucket.MyBucket

# Export state for reference
tofu state pull > current-state.json
```

## Phase 3: Translate CDKTF TypeScript to HCL

Convert CDKTF code to equivalent HCL.

```typescript
// CDKTF TypeScript (original)
import { S3Bucket } from "@cdktf/provider-aws/lib/s3-bucket";
import { S3BucketVersioningA } from "@cdktf/provider-aws/lib/s3-bucket-versioning";

const bucket = new S3Bucket(this, "AppBucket", {
  bucket: "my-app-bucket",
  tags: { Environment: "prod", ManagedBy: "cdktf" },
});

new S3BucketVersioningA(this, "AppBucketVersioning", {
  bucket: bucket.id,
  versioningConfiguration: { status: "Enabled" },
});
```

```hcl
# OpenTofu HCL equivalent
resource "aws_s3_bucket" "app" {
  bucket = "my-app-bucket"

  tags = {
    Environment = "prod"
    ManagedBy   = "opentofu"
  }
}

resource "aws_s3_bucket_versioning" "app" {
  bucket = aws_s3_bucket.app.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

## Phase 4: Handle CDKTF State Addressing

CDKTF uses different resource addresses than standard HCL. You need to handle this during migration.

```bash
# CDKTF state addresses use the construct ID (capitalized)
# aws_s3_bucket.AppBucket

# Standard OpenTofu HCL addresses use snake_case
# aws_s3_bucket.app

# When importing, use the cloud resource ID (not the state address)
```

```hcl
# imports.tf - use cloud resource IDs, not CDKTF state addresses

import {
  id = "my-app-bucket"  # S3 bucket name
  to = aws_s3_bucket.app
}

import {
  id = "vpc-0abc12345def67890"  # VPC ID from AWS
  to = aws_vpc.main
}
```

## Phase 5: Migrate State

Move from the CDKTF-managed state to a standard OpenTofu state.

```bash
# Option 1: Create new OpenTofu root and import all resources
# (recommended for clean migration)
mkdir opentofu-migration
cd opentofu-migration
# Write your HCL config
# Add import blocks
tofu init
tofu apply  # imports resources into new state

# Option 2: Copy state and rename resources (advanced)
# Copy the CDKTF state file to a new location
cp cdktf.out/stacks/my-stack/terraform.tfstate ./
# Use tofu state mv to rename addresses
tofu state mv 'aws_s3_bucket.AppBucket' 'aws_s3_bucket.app'
```

## Phase 6: Decommission CDKTF

After validating the OpenTofu migration.

```bash
# Verify no changes in OpenTofu plan
tofu plan  # No changes expected

# Archive CDKTF code
git mv src/stacks src/stacks-archived
git commit -m "Archive CDKTF code - migrated to OpenTofu HCL"

# Remove CDKTF dependencies
npm uninstall cdktf cdktf-cli
```

## Summary

Migrating from CDKTF to OpenTofu HCL uses CDKTF's synthesized JSON as a translation reference. The key challenge is handling different resource address naming conventions between CDKTF (which uses TypeScript class names) and HCL (which uses snake_case). The safest approach is to import all resources fresh into a new OpenTofu state rather than trying to rename state addresses. After migration, you eliminate the synthesis build step and make your infrastructure configuration directly readable by the entire team.
