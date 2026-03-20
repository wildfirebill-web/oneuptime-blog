# How to Import Multiple Resources at Once in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn strategies for importing multiple existing cloud resources into OpenTofu state efficiently using batch import blocks, for_each, and config generation.

## Introduction

Brownfield infrastructure adoption often requires importing dozens or hundreds of existing resources. OpenTofu provides several strategies for efficient bulk imports: multiple import blocks, `for_each` import blocks, and config generation. This guide covers a practical multi-resource import workflow.

## Strategy 1: Multiple Import Blocks

For a small number of resources:

```hcl
# imports.tf

import { to = aws_vpc.main      id = "vpc-0a1b2c3d" }
import { to = aws_subnet.pub1   id = "subnet-0a1b2c" }
import { to = aws_subnet.pub2   id = "subnet-0d4e5f" }
import { to = aws_subnet.priv1  id = "subnet-0g7h8i" }
import { to = aws_subnet.priv2  id = "subnet-0j9k0l" }
import { to = aws_internet_gateway.main id = "igw-0a1b2c3d" }
import { to = aws_nat_gateway.main id = "nat-0a1b2c3d4e5f6789" }
```

## Strategy 2: for_each Import Blocks

For same-type resources at scale:

```hcl
locals {
  subnets = {
    "pub-1"  = "subnet-0a1b2c3d"
    "pub-2"  = "subnet-0e4f5a6b"
    "priv-1" = "subnet-0c7d8e9f"
    "priv-2" = "subnet-0g1h2i3j"
  }
}

import {
  for_each = local.subnets
  to       = aws_subnet.main[each.key]
  id       = each.value
}
```

## Strategy 3: Comprehensive Environment Import

Importing a complete environment at once:

```hcl
# full-environment-import.tf

locals {
  vpc_id = "vpc-0a1b2c3d4e5f6789"

  # All resources to import
  instances = {
    "app-1"  = "i-0000000000000001"
    "app-2"  = "i-0000000000000002"
    "db-1"   = "i-0000000000000003"
  }

  security_groups = {
    "app"      = "sg-0000000000000001"
    "database" = "sg-0000000000000002"
    "lb"       = "sg-0000000000000003"
  }

  rds_instances = {
    "primary"  = "prod-db-primary"
    "replica"  = "prod-db-replica"
  }
}

# Import VPC
import {
  to = aws_vpc.main
  id = local.vpc_id
}

# Import EC2 instances
import {
  for_each = local.instances
  to       = aws_instance.app[each.key]
  id       = each.value
}

# Import security groups
import {
  for_each = local.security_groups
  to       = aws_security_group.main[each.key]
  id       = each.value
}

# Import RDS instances
import {
  for_each = local.rds_instances
  to       = aws_db_instance.main[each.key]
  id       = each.value
}
```

## Automated Import ID Discovery

Script to discover resource IDs for import:

```bash
#!/bin/bash
# discover-resources.sh

# Get all EC2 instances with their names
echo "EC2 Instances:"
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value | [0]]' \
  --output text | while read id name; do
  echo "  \"${name:-$id}\" = \"$id\""
done

# Get all S3 buckets
echo "S3 Buckets:"
aws s3api list-buckets --query 'Buckets[*].Name' --output text | tr '\t' '\n' | while read bucket; do
  echo "  \"$bucket\" = \"$bucket\""
done

# Get all RDS instances
echo "RDS Instances:"
aws rds describe-db-instances \
  --query 'DBInstances[*].DBInstanceIdentifier' \
  --output text | tr '\t' '\n' | while read id; do
  echo "  \"$id\" = \"$id\""
done
```

## Using Config Generation for Multiple Resources

```bash
# Add all import blocks, then generate config for all at once
tofu plan -generate-config-out=imported-resources.tf

# Review and clean up the generated file
wc -l imported-resources.tf  # Check size

# Apply all imports in one step
tofu apply
```

## Conclusion

Bulk importing resources in OpenTofu is efficient with `for_each` import blocks and config generation. Start by discovering resource IDs with cloud CLI tools, then use `for_each` imports for same-type resources and individual import blocks for unique resources. The `-generate-config-out` flag handles config creation. After import, always run `tofu plan` to verify all resources reconcile without changes.
