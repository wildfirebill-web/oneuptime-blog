# How to Generate Configuration After Import in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Import, Config Generation, generate-config-out, HCL

Description: Learn how to use OpenTofu's -generate-config-out flag to automatically generate HCL configuration from imported resources, accelerating the migration of existing infrastructure to code.

## Introduction

Writing HCL to match existing resources before importing is tedious and error-prone. OpenTofu's `-generate-config-out` flag generates HCL from `import` blocks automatically, providing a starting point that you refine rather than write from scratch.

## Basic Usage

```hcl
# Step 1: Write just the import blocks (no resource config needed yet)
# imports.tf
import {
  to = aws_vpc.main
  id = "vpc-0123456789abcdef0"
}

import {
  to = aws_subnet.public["us-east-1a"]
  id = "subnet-0123456789abcdef0"
}

import {
  to = aws_security_group.app
  id = "sg-0123456789abcdef0"
}
```

```bash
# Step 2: Generate HCL configuration for all import blocks
tofu plan -generate-config-out=generated_resources.tf

# This creates generated_resources.tf with HCL for all imported resources
```

## Understanding Generated Output

The generated file contains complete resource blocks:

```hcl
# generated_resources.tf (auto-generated - do not edit directly)
resource "aws_vpc" "main" {
  assign_generated_ipv6_cidr_block     = false
  cidr_block                           = "10.0.0.0/16"
  enable_dns_hostnames                 = true
  enable_dns_support                   = true
  enable_network_address_usage_metrics = false
  instance_tenancy                     = "default"
  tags = {
    "Name"        = "prod-vpc"
    "Environment" = "prod"
  }
}
```

## Cleaning Up Generated Configuration

The generated config includes all attributes including defaults. Clean it up:

```hcl
# Before cleanup (generated):
resource "aws_vpc" "main" {
  assign_generated_ipv6_cidr_block     = false   # Remove - this is the default
  cidr_block                           = "10.0.0.0/16"
  enable_dns_hostnames                 = true
  enable_dns_support                   = true
  enable_network_address_usage_metrics = false   # Remove - this is the default
  instance_tenancy                     = "default"  # Remove - this is the default
  tags = {
    "Name"        = "prod-vpc"
    "Environment" = "prod"
  }
}

# After cleanup (curated):
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name        = "prod-vpc"
    Environment = "prod"
  }
}
```

## Complete Import Workflow with Config Generation

```bash
#!/bin/bash
# complete_import_workflow.sh

echo "Step 1: Write import blocks"
cat > imports.tf << 'EOF'
import { to = aws_vpc.main; id = "vpc-0123456789abcdef0" }
import { to = aws_internet_gateway.main; id = "igw-0123456789abcdef0" }
EOF

echo "Step 2: Generate configuration"
tofu plan -generate-config-out=generated.tf

echo "Step 3: Review generated config"
cat generated.tf

echo "Step 4: Apply the import"
# After reviewing and cleaning up generated.tf:
tofu apply

echo "Step 5: Remove import blocks (they're no longer needed)"
rm imports.tf

echo "Step 6: Verify clean state"
tofu plan
# Should show: No changes. Infrastructure is up-to-date.
```

## Handling for_each Imports with Generated Config

```hcl
# Import blocks for for_each resources
import {
  to = aws_subnet.private["us-east-1a"]
  id = "subnet-abc"
}

import {
  to = aws_subnet.private["us-east-1b"]
  id = "subnet-def"
}

# The generated config will include a single for_each resource block
# You'll need to refactor it from individual blocks to a for_each pattern
```

## Limitations and Caveats

```hcl
# Generated config may include sensitive values in plaintext
# Review and remove before committing to version control

# Also check for deprecated attributes in generated config
# The generator uses current provider schema, not future-compatible HCL

# Some computed-only attributes will appear in generated config
# They don't belong in HCL and should be removed
resource "aws_vpc" "main" {
  # Remove these computed attributes:
  # arn     = "arn:aws:ec2:us-east-1:123:vpc/vpc-abc"  # Computed
  # id      = "vpc-0123456789abcdef0"                   # Computed
  # main_route_table_id = "rtb-abc"                     # Computed

  # Keep only configuration attributes:
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
}
```

## Conclusion

The `-generate-config-out` feature transforms import from a "write-then-verify" workflow to a "generate-then-refine" workflow. Use it for bulk imports to get 80% of the HCL written automatically, then spend your time cleaning up defaults and restructuring for reuse. Always review generated configs for sensitive values and computed attributes before committing.
