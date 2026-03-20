# How to Migrate AWS Infrastructure from CloudFormation to OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, CloudFormation, Migration, AWS, Infrastructure as Code

Description: Learn how to migrate existing AWS infrastructure managed by CloudFormation stacks into OpenTofu state without recreating resources.

## Introduction

Migrating from CloudFormation to OpenTofu involves three phases: writing OpenTofu configuration that matches your existing resources, importing those resources into state, and decommissioning the CloudFormation stacks. The key principle is to import without destroying - your running infrastructure should never be interrupted.

## Phase 1: Audit Your CloudFormation Stacks

Start by inventorying what you have.

```bash
# List all CloudFormation stacks

aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[*].[StackName,StackStatus]' \
  --output table

# Get detailed resources for a specific stack
aws cloudformation list-stack-resources \
  --stack-name my-app-stack \
  --query 'StackResourceSummaries[*].[ResourceType,LogicalResourceId,PhysicalResourceId]' \
  --output table

# Export the template for reference
aws cloudformation get-template \
  --stack-name my-app-stack \
  --query 'TemplateBody' > my-app-stack-template.json
```

## Phase 2: Write OpenTofu Configuration

Translate CloudFormation resources into HCL. Use the AWS provider documentation for the correct resource types and arguments.

```hcl
# CloudFormation (original)
# Type: AWS::S3::Bucket
# Properties:
#   BucketName: my-app-data
#   VersioningConfiguration:
#     Status: Enabled

# OpenTofu equivalent
resource "aws_s3_bucket" "app_data" {
  bucket = "my-app-data"
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

## Phase 3: Import Existing Resources

Use import blocks to bring existing resources into OpenTofu state.

```hcl
# imports.tf
import {
  id = "my-app-data"
  to = aws_s3_bucket.app_data
}

import {
  id = "my-app-vpc"
  to = aws_vpc.main
}

import {
  id = "sg-0abc12345def67890"
  to = aws_security_group.app
}
```

```bash
# Preview the import
tofu plan

# If the plan shows no unexpected changes, apply the imports
tofu apply

# Remove the import blocks after successful import
# Import blocks are only needed once
```

## Phase 4: Handle CloudFormation-Specific Constructs

Some CloudFormation features need translation.

```hcl
CloudFormation → OpenTofu equivalents:

Fn::GetAtt      → resource attribute references (resource.name.attribute)
Ref             → resource.name.id or var.name
Parameters      → variable blocks
Outputs         → output blocks
Conditions      → locals with conditional expressions
DependsOn       → depends_on meta-argument
Mappings        → local maps or variables
AWS::NoValue    → null
```

```hcl
# CloudFormation Fn::Sub substitution
# "arn:aws:s3:::${BucketName}/*"

# OpenTofu equivalent
"arn:aws:s3:::${aws_s3_bucket.app_data.bucket}/*"
```

## Phase 5: Validate and Decommission

Verify OpenTofu manages the resources correctly before removing CloudFormation.

```bash
# Run plan - should show no changes if config matches reality
tofu plan

# If plan is clean, disable stack termination protection
aws cloudformation update-termination-protection \
  --stack-name my-app-stack \
  --no-enable-termination-protection

# Delete the CloudFormation stack with RETAIN on all resources
# This removes CF management without deleting resources
aws cloudformation delete-stack \
  --stack-name my-app-stack \
  --retain-resources LogicalResourceId1 LogicalResourceId2
```

## Handling Nested Stacks

For stacks with nested stacks, migrate leaf stacks first.

```bash
# List nested stacks
aws cloudformation list-stack-resources \
  --stack-name parent-stack \
  --query 'StackResourceSummaries[?ResourceType==`AWS::CloudFormation::Stack`]'

# Migrate child stacks first, then the parent
# Decommission in reverse order of creation
```

## Summary

Migrating from CloudFormation to OpenTofu requires careful planning but does not require any downtime. The process: audit all stacks, write HCL configuration that matches the existing resources, use import blocks to pull them into state, validate with `tofu plan` (should show no changes), then delete CloudFormation stacks with resource retention. The ability to use `tofu plan -generate-config-out` to auto-generate HCL from import blocks can significantly accelerate the initial configuration writing step.
