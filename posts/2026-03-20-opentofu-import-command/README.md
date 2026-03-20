# How to Use tofu import to Import Existing Resources

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use tofu import to bring existing infrastructure resources under OpenTofu management without recreating them.

## Introduction

`tofu import` adds an existing cloud resource to the OpenTofu state file so it can be managed going forward. After importing, you must write the matching configuration - OpenTofu does not generate configuration from the import. The modern declarative approach using `import` blocks is generally preferred, but the CLI command remains useful for quick, one-off imports.

## Basic Syntax

```bash
tofu import <resource_address> <resource_id>
```

## Import an S3 Bucket

```bash
# Write the configuration first

# resource "aws_s3_bucket" "existing" {
#   bucket = "my-existing-bucket"
# }

# Then import using the bucket name as the ID
tofu import aws_s3_bucket.existing my-existing-bucket
```

## Import an EC2 Instance

```bash
tofu import aws_instance.web i-0abc123def456789
```

## Import an IAM Role

```bash
tofu import aws_iam_role.app my-existing-role-name
```

## Import into a Module

```bash
# Import into a resource inside a module
tofu import module.networking.aws_vpc.main vpc-0abc123456
```

## Import a Resource with count

```bash
# Import the first instance (index 0)
tofu import 'aws_instance.web[0]' i-0abc123def456789
```

## Import a for_each Resource

```bash
# Import a resource with a string key
tofu import 'aws_s3_bucket.buckets["production"]' production-bucket-name
```

## Finding the Import ID Format

Each resource type uses a different ID format. Check the provider documentation:

```bash
# Check what the import ID format is for a resource
# Example from AWS provider docs:
# aws_s3_bucket: bucket name
# aws_iam_role: role name
# aws_instance: instance ID (i-...)
# aws_vpc: VPC ID (vpc-...)
# aws_subnet: subnet ID (subnet-...)
```

## The Import Workflow

```bash
# Step 1: Write the resource configuration
cat > main.tf << 'EOF'
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}
EOF

# Step 2: Import the resource into state
tofu import aws_s3_bucket.existing my-existing-bucket

# Step 3: Run plan to see if config matches state
tofu plan
# If there are differences, update main.tf to match

# Step 4: Apply to reconcile any differences
tofu apply
```

## Verify After Import

```bash
# Check the imported resource appears in state
tofu state list
tofu state show aws_s3_bucket.existing
```

## Dry-Run Import (modern approach)

The `import` CLI command immediately modifies state. For a safer approach, use import blocks which work through the plan/apply workflow:

```hcl
# import.tf
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}
```

```bash
tofu plan    # Preview the import
tofu apply   # Perform the import
```

## Common Import ID Formats

| Resource | Import ID |
|---|---|
| `aws_s3_bucket` | bucket name |
| `aws_instance` | instance ID (`i-...`) |
| `aws_vpc` | VPC ID (`vpc-...`) |
| `aws_iam_role` | role name |
| `google_storage_bucket` | `project/bucket-name` |
| `azurerm_resource_group` | full resource ID path |

## Conclusion

`tofu import` brings existing resources under OpenTofu management. Write the configuration first, import the resource, then reconcile any differences between configuration and imported state by updating the configuration until `tofu plan` shows no changes. For importing many resources or for team workflows that need review before committing, prefer the declarative `import` block approach which goes through the normal plan/apply workflow.
