# How to Migrate Infrastructure from Pulumi to OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Pulumi, Migration, Infrastructure as Code, State Management

Description: Learn how to migrate infrastructure managed by Pulumi into OpenTofu state without recreating cloud resources.

## Introduction

Pulumi uses general-purpose programming languages (TypeScript, Python, Go) to define infrastructure. OpenTofu uses HCL, which many operators prefer for its declarative clarity. Migrating from Pulumi to OpenTofu involves exporting resource information from Pulumi's state, writing matching HCL configurations, and importing resources into OpenTofu state.

## Phase 1: Export Pulumi State

Start by understanding what Pulumi manages.

```bash
# Export the Pulumi stack state to JSON
pulumi stack export --stack prod > pulumi-state.json

# View a summary of resources in the stack
pulumi stack --stack prod --show-urns

# List all resources with their types and names
cat pulumi-state.json | jq '.deployment.resources[] | {type: .type, urn: .urn, id: .id}'
```

## Phase 2: Map Pulumi Resources to OpenTofu

Extract resource information from the Pulumi state.

```bash
# Extract AWS resource IDs from Pulumi state
cat pulumi-state.json | jq -r '
  .deployment.resources[]
  | select(.type | startswith("aws:"))
  | [.type, .id]
  | @tsv
'

# Example output:
# aws:s3/bucket:Bucket     my-app-bucket
# aws:ec2/vpc:Vpc          vpc-0abc12345
# aws:rds/instance:Instance myapp-db
```

## Phase 3: Write OpenTofu Configuration

Translate Pulumi code to HCL. The underlying cloud resources are the same.

```typescript
// Pulumi TypeScript (original)
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("app-bucket", {
  bucket: "my-app-bucket",
  versioning: {
    enabled: true,
  },
  tags: {
    Environment: "prod",
    ManagedBy: "pulumi",
  },
});
```

```hcl
# OpenTofu equivalent
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

## Phase 4: Import Resources into OpenTofu

Use import blocks with the resource IDs extracted from Pulumi state.

```hcl
# imports.tf

import {
  id = "my-app-bucket"
  to = aws_s3_bucket.app
}

import {
  id = "vpc-0abc12345def67890"
  to = aws_vpc.main
}

import {
  id = "myapp-prod-db"
  to = aws_db_instance.main
}
```

```bash
# Set up OpenTofu backend first
tofu init

# Preview the import - should show no destructive changes
tofu plan

# Apply the import
tofu apply

# Remove import blocks after successful import
```

## Phase 5: Handle Pulumi-Specific Patterns

Some Pulumi patterns require careful translation.

```typescript
// Pulumi component resource (like a module)
class WebApp extends pulumi.ComponentResource {
  public url: pulumi.Output<string>;

  constructor(name: string, args: WebAppArgs) {
    super("custom:web:WebApp", name, {});
    // creates multiple resources
  }
}
```

```hcl
# OpenTofu module equivalent
module "web_app" {
  source   = "./modules/web-app"
  name     = var.app_name
  environment = var.environment
}
```

## Phase 6: Decommission Pulumi Stack

After validating OpenTofu manages the resources correctly.

```bash
# Verify OpenTofu plan shows no changes
tofu plan  # No changes expected

# Remove resources from Pulumi state without destroying them
# Use pulumi state delete for each resource
pulumi state delete --stack prod "urn:pulumi:prod::myapp::aws:s3/bucket:Bucket::app-bucket"

# Or destroy the Pulumi stack records (not the cloud resources)
# This requires removing each resource from state manually

# Archive your Pulumi code
git mv pulumi/ pulumi-archived/
git commit -m "Archive Pulumi code - migrated to OpenTofu"
```

## Summary

Migrating from Pulumi to OpenTofu follows the standard import workflow: export Pulumi state to understand existing resource IDs, write HCL configuration that matches your current infrastructure, import resources into OpenTofu state, validate with `tofu plan` (no changes expected), and remove resources from Pulumi state. The cloud resources themselves are never touched during migration — only the management layer changes.
