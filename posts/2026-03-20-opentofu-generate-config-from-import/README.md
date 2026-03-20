# How to Generate Configuration from Imported Resources in OpenTofu (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Import

Description: Learn how to use OpenTofu's -generate-config-out flag to automatically generate resource configuration from existing infrastructure during import.

## Introduction

Writing resource configuration manually before importing is tedious for complex resources. OpenTofu's `-generate-config-out` flag automates this - it reads the actual resource from the cloud provider and generates a complete HCL configuration file. This is particularly useful when importing resources with many attributes, such as EKS clusters, RDS instances, or complex IAM policies.

## Basic Usage

```hcl
# import.tf - just the import block, no resource block yet

import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}
```

```bash
# Generate the resource configuration automatically
tofu plan -generate-config-out=generated.tf
```

This creates `generated.tf` with the full resource configuration read from the cloud provider.

## Generated Output Example

```bash
# generated.tf (auto-generated - do not edit directly)
resource "aws_s3_bucket" "existing" {
  bucket        = "my-existing-bucket"
  force_destroy = false

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}
```

## Workflow

```bash
# Step 1: Write the import block
cat > import.tf << 'EOF'
import {
  to = aws_eks_cluster.main
  id = "production-cluster"
}
EOF

# Step 2: Generate configuration
tofu plan -generate-config-out=generated.tf

# Step 3: Review and clean up generated.tf
# Remove read-only attributes and add variables as needed

# Step 4: Apply to import the resource
tofu apply

# Step 5: Remove the import block from import.tf
# Step 6: Integrate generated.tf into your main configuration
```

## Cleaning Up Generated Configuration

Generated configuration often includes computed/read-only attributes that should be removed:

```hcl
# Before cleanup (generated output - includes computed values)
resource "aws_s3_bucket" "existing" {
  bucket                      = "my-existing-bucket"
  bucket_domain_name          = "my-existing-bucket.s3.amazonaws.com"  # Remove - computed
  bucket_regional_domain_name = "my-existing-bucket.s3.us-east-1.amazonaws.com"  # Remove - computed
  id                          = "my-existing-bucket"  # Remove - computed
  region                      = "us-east-1"  # May remove or keep
  force_destroy               = false
}
```

```hcl
# After cleanup
resource "aws_s3_bucket" "existing" {
  bucket        = "my-existing-bucket"
  force_destroy = false
}
```

## Generating Config for Multiple Resources

```hcl
# import.tf - multiple import blocks
import {
  to = aws_s3_bucket.data
  id = "acme-data-bucket"
}

import {
  to = aws_s3_bucket.logs
  id = "acme-logs-bucket"
}
```

```bash
# Generates configuration for all blocks in one file
tofu plan -generate-config-out=generated.tf
```

## Limitations

- Generated files cannot be applied with `-generate-config-out` - only for generation
- You must apply separately after reviewing the generated output
- Some attributes may need manual adjustment (e.g., replacing hardcoded values with variables)
- The generated file may include deprecated attributes that should be removed

```bash
# Cannot apply and generate in one step
# This generates, then you review, then you apply separately
tofu plan -generate-config-out=generated.tf
# Review generated.tf
tofu apply  # Apply after reviewing
```

## Best Practices

```bash
# 1. Always review generated output before applying
# 2. Parameterize hardcoded values into variables
# 3. Remove computed/read-only attributes
# 4. Move generated code into appropriate modules

# Example: parameterize after generation
# Change: bucket = "acme-data-prod-1234567890"
# To:     bucket = var.data_bucket_name
```

## Conclusion

`-generate-config-out` eliminates the tedious work of writing resource configurations from scratch when importing existing infrastructure. Write just the import block, generate the configuration, review and clean it up (remove computed attributes, parameterize hardcoded values), then apply. This dramatically speeds up adoption of OpenTofu for existing infrastructure.
