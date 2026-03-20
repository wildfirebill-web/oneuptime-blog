# How to Define Local Values in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Locals, HCL, Infrastructure as Code, DevOps

Description: A guide to defining and using local values in OpenTofu to reduce repetition and simplify complex expressions.

## Introduction

Local values in OpenTofu are named expressions that can be referenced multiple times within a module. They are similar to local variables in programming languages - they compute a value once and allow you to reference it by name throughout the configuration.

## Basic Local Definition

```hcl
# locals.tf

locals {
  # Simple string
  environment = "production"

  # Concatenation
  name_prefix = "${var.project_name}-${var.environment}"

  # Computation
  replica_count = var.environment == "prod" ? 3 : 1

  # Tag map (used across multiple resources)
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "OpenTofu"
    Owner       = "platform-team"
    Repository  = "github.com/myorg/infrastructure"
  }
}
```

## Types of Local Values

```hcl
locals {
  # String
  resource_prefix = "${var.project}-${var.env}"

  # Number
  web_instance_count = var.environment == "prod" ? 4 : 1

  # Boolean
  is_production = var.environment == "prod"

  # List
  all_az_suffixes = ["a", "b", "c"]

  # Map
  instance_type_map = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }

  # Computed map
  instance_type = local.instance_type_map[var.environment]

  # List computed from count
  az_list = [
    for i in range(var.az_count) :
    "${var.aws_region}${local.all_az_suffixes[i]}"
  ]
}
```

## Common Patterns

Resource Naming

```hcl
locals {
  # Consistent naming convention
  vpc_name       = "${local.name_prefix}-vpc"
  alb_name       = "${local.name_prefix}-alb"
  ecs_cluster_name = "${local.name_prefix}-cluster"
  rds_identifier = "${local.name_prefix}-db"

  # Add suffix for uniqueness
  bucket_name = "${local.name_prefix}-state-${random_id.suffix.hex}"
}
```

### Conditional Configuration

```hcl
locals {
  is_production = var.environment == "prod"
  is_staging    = var.environment == "staging"

  # Environment-specific settings
  db_multi_az        = local.is_production
  db_backup_days     = local.is_production ? 30 : local.is_staging ? 7 : 1
  deletion_protection = local.is_production

  # Instance sizing
  web_instance_type = local.is_production ? "t3.large" : "t3.micro"
  db_instance_class = local.is_production ? "db.r5.large" : "db.t3.micro"
}
```

### Computed CIDR Blocks

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_count" {
  type    = number
  default = 3
}

locals {
  # Generate subnet CIDRs automatically
  public_subnet_cidrs = [
    for i in range(var.subnet_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
  # Result: ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]

  private_subnet_cidrs = [
    for i in range(var.subnet_count) :
    cidrsubnet(var.vpc_cidr, 8, i + 10)  # Offset to avoid overlap
  ]
  # Result: ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
}
```

## Referencing Local Values

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Environment = var.environment
    ManagedBy   = "OpenTofu"
  }
}

# Reference with local.<name>

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = merge(
    local.common_tags,           # Reference local
    {
      Name = "${local.name_prefix}-vpc"  # Interpolate local
    }
  )
}

resource "aws_subnet" "public" {
  count  = length(var.public_subnet_cidrs)
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-public-${count.index + 1}"
    }
  )
}
```

## Conclusion

Local values are one of the most useful tools in OpenTofu for eliminating repetition and making configurations more readable. By computing values once in a `locals` block - like tag maps, naming prefixes, and environment-based selections - you ensure consistency across all resources that use those values. Any change in a local value automatically propagates to all resources that reference it.
