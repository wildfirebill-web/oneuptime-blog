# How to Generate Configuration from Imported Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use OpenTofu's -generate-config-out flag to automatically generate HCL configuration for imported resources, accelerating brownfield adoption.

## Introduction

OpenTofu 1.6 introduced the `-generate-config-out` flag for `tofu plan`. When used with import blocks, it automatically generates the HCL resource configuration based on the actual imported resource's attributes. This dramatically speeds up brownfield infrastructure adoption - you don't need to manually write every resource configuration.

## Basic Usage

### Step 1: Write Only the Import Block

```hcl
# imports.tf - just the import block, no resource block required

import {
  to = aws_vpc.main
  id = "vpc-0a1b2c3d4e5f6789"
}

import {
  to = aws_instance.web
  id = "i-0123456789abcdef0"
}
```

### Step 2: Generate Configuration

```bash
# Generate HCL configuration for all imported resources
tofu plan -generate-config-out=generated.tf

# Output:
# aws_vpc.main: Preparing import... [id=vpc-0a1b2c3d4e5f6789]
# aws_instance.web: Preparing import... [id=i-0123456789abcdef0]
# OpenTofu has generated configuration for the following resources:
#   - aws_vpc.main in generated.tf
#   - aws_instance.web in generated.tf
```

### Step 3: Review and Clean Up Generated Configuration

The generated file (`generated.tf`) contains all the resource attributes:

```hcl
# generated.tf (auto-generated, needs cleanup)
resource "aws_vpc" "main" {
  assign_generated_ipv6_cidr_block     = false
  cidr_block                           = "10.0.0.0/16"
  enable_classiclink                   = null
  enable_classiclink_dns_support       = null
  enable_dns_hostnames                 = true
  enable_dns_support                   = true
  id                                   = "vpc-0a1b2c3d4e5f6789"
  instance_tenancy                     = "default"
  ipv4_ipam_pool_id                    = null
  ipv4_netmask_length                  = null
  tags = {
    "Environment" = "production"
    "Name"        = "main-vpc"
    "ManagedBy"   = "terraform"
  }
}
```

Clean up the generated configuration:

```hcl
# cleaned-up resources.tf
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Environment = "production"
    Name        = "main-vpc"
    ManagedBy   = "opentofu"
  }
}
```

### Step 4: Apply the Import

```bash
tofu apply
# aws_vpc.main: Importing...
# aws_instance.web: Importing...
# Import complete!
```

### Step 5: Verify and Remove Import Blocks

```bash
tofu plan
# No changes. Infrastructure is up-to-date.
```

Remove import blocks from `imports.tf`.

## Generated Config for Complex Resources

For complex resources like EKS clusters:

```hcl
# imports.tf
import {
  to = aws_eks_cluster.main
  id = "my-eks-cluster"
}
```

```bash
tofu plan -generate-config-out=eks-generated.tf
# Generates ALL EKS cluster attributes including nested blocks
```

## Batch Generation with for_each

```hcl
locals {
  buckets = {
    "assets"  = "company-assets"
    "logs"    = "company-logs"
  }
}

import {
  for_each = local.buckets
  to       = aws_s3_bucket.main[each.key]
  id       = each.value
}
```

```bash
# Generates configuration for all buckets
tofu plan -generate-config-out=buckets-generated.tf
```

## Limitations

- Generated config may include deprecated attributes
- Not all attributes need to be specified (many have defaults)
- Review and simplify the generated config before committing

## Conclusion

The `-generate-config-out` flag eliminates the most tedious part of brownfield IaC adoption - manually writing resource configurations for every existing resource. Use it to bootstrap your OpenTofu configurations, then clean up the generated code to remove unnecessary attributes, improve readability, and ensure the configuration reflects your desired state rather than just the current state.
