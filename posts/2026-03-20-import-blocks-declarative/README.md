# How to Use Import Blocks for Declarative Import in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use OpenTofu import blocks to declaratively import existing cloud resources into your state, enabling code-reviewed and version-controlled imports.

## Introduction

OpenTofu 1.5 introduced `import` blocks - a declarative way to import existing cloud resources. Unlike the imperative `tofu import` CLI command, import blocks live in your configuration files, making imports reviewable, version-controlled, and repeatable.

## Basic Import Block Syntax

```hcl
# Import an existing EC2 instance

import {
  to = aws_instance.web
  id = "i-0123456789abcdef0"
}

# Import an existing VPC
import {
  to = aws_vpc.main
  id = "vpc-0a1b2c3d4e5f6789"
}
```

## Complete Import Workflow

### Step 1: Identify Resources to Import

```bash
# Find the resource IDs from AWS CLI
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key=="Name"].Value]'

# VPC
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,Tags[?Key=="Name"].Value]'
```

### Step 2: Write Import Blocks and Resource Configurations

```hcl
# imports.tf
import {
  to = aws_vpc.production
  id = "vpc-0a1b2c3d4e5f6789"
}

import {
  to = aws_instance.web
  id = "i-0123456789abcdef0"
}

# resources.tf - write the resource configuration
resource "aws_vpc" "production" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "production"
    Environment = "production"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.public.id

  tags = {
    Name = "web-server"
  }
}
```

### Step 3: Plan the Import

```bash
tofu plan

# Output shows imports:
# aws_vpc.production will be imported
# aws_instance.web will be imported
```

### Step 4: Apply to Execute the Import

```bash
tofu apply

# Output:
# aws_vpc.production: Importing... [id=vpc-0a1b2c3d4e5f6789]
# aws_vpc.production: Import complete [id=vpc-0a1b2c3d4e5f6789]
# aws_instance.web: Importing... [id=i-0123456789abcdef0]
# aws_instance.web: Import complete [id=i-0123456789abcdef0]
```

### Step 5: Remove Import Blocks

After a successful import, remove the import blocks (they're one-time operations):

```hcl
# imports.tf - remove after successful import
# import {
#   to = aws_vpc.production
#   id = "vpc-0a1b2c3d4e5f6789"
# }
```

Then run `tofu plan` - it should show no changes.

## Import Block with for_each (OpenTofu 1.7+)

```hcl
# Import multiple resources using for_each
locals {
  instances = {
    "web-1" = "i-0123456789abcdef0"
    "web-2" = "i-abcdef0123456789"
  }
}

import {
  for_each = local.instances
  to       = aws_instance.web[each.key]
  id       = each.value
}

resource "aws_instance" "web" {
  for_each = local.instances

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}
```

## Advantages of Import Blocks over CLI Import

| Feature | CLI import | Import blocks |
|---------|-----------|---------------|
| Version controlled | No | Yes |
| Code reviewable | No | Yes |
| Repeatable | No (already imported) | Yes (idempotent) |
| for_each support | No | Yes (1.7+) |
| Config generation | No | Yes (1.6+) |

## Conclusion

Import blocks transform resource imports from one-off CLI operations into code-reviewed, version-controlled infrastructure changes. Use them as part of a standard PR workflow to bring existing resources under OpenTofu management. After the import is applied and verified, remove the import blocks from your configuration - they've served their purpose.
