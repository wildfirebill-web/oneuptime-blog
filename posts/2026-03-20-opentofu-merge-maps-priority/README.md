# How to Merge Multiple Maps with Priority in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, merge, Maps, Functions, Configuration

Description: Learn how to merge multiple maps with priority ordering in OpenTofu using the merge function, enabling layered configuration defaults with environment and service-level overrides.

## Overview

The `merge` function in OpenTofu combines multiple maps into one, with later arguments taking priority over earlier ones. This makes it ideal for implementing layered configuration systems where defaults flow from global → environment → service level, with each layer able to override the previous.

## Basic merge Behavior

```hcl
locals {
  defaults = {
    instance_type    = "t3.micro"
    disk_size_gb     = 20
    monitoring       = false
    deletion_protection = false
  }

  production_overrides = {
    instance_type       = "t3.large"
    monitoring          = true
    deletion_protection = true
  }

  # merge: later maps win on key conflicts
  prod_config = merge(local.defaults, local.production_overrides)
  # {
  #   instance_type       = "t3.large"   (from production_overrides)
  #   disk_size_gb        = 20           (from defaults - not overridden)
  #   monitoring          = true         (from production_overrides)
  #   deletion_protection = true         (from production_overrides)
  # }
}
```

## Three-Layer Configuration: Global → Environment → Service

```hcl
# Global defaults applied to every resource
locals {
  global_defaults = {
    instance_type       = "t3.micro"
    min_capacity        = 1
    max_capacity        = 3
    disk_size_gb        = 20
    enable_monitoring   = false
    deletion_protection = false
    backup_retention    = 7
    log_level           = "warn"
    multi_az            = false
  }

  # Environment-level overrides
  env_overrides = {
    prod = {
      instance_type       = "t3.large"
      min_capacity        = 2
      max_capacity        = 10
      enable_monitoring   = true
      deletion_protection = true
      backup_retention    = 30
      log_level           = "error"
      multi_az            = true
    }
    staging = {
      instance_type    = "t3.medium"
      backup_retention = 14
      log_level        = "info"
    }
    dev = {
      log_level = "debug"
    }
  }

  # Service-specific overrides
  service_overrides = {
    api = {
      max_capacity = 20  # API needs more headroom
      disk_size_gb = 50
    }
    worker = {
      instance_type = "c5.xlarge"  # CPU-optimized for workers
    }
    db = {
      instance_type       = "r5.large"  # Memory-optimized for DB
      deletion_protection = true        # Always protect DB
      backup_retention    = 90
    }
  }
}

# Merge all three layers for a specific env/service combination
module "service_config" {
  source = "./modules/service"

  for_each = {
    "prod/api"    = merge(local.global_defaults, local.env_overrides["prod"],    local.service_overrides["api"])
    "prod/worker" = merge(local.global_defaults, local.env_overrides["prod"],    local.service_overrides["worker"])
    "staging/api" = merge(local.global_defaults, local.env_overrides["staging"], local.service_overrides["api"])
  }

  config = each.value
}
```

## Dynamic Merge with Variable Inputs

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

variable "service_name" {
  type    = string
  default = "api"
}

variable "custom_config" {
  type    = map(any)
  default = {}
  description = "Caller-provided overrides with highest priority"
}

locals {
  base_tags = {
    ManagedBy   = "OpenTofu"
    Environment = var.environment
  }

  env_tags = {
    prod = {
      CostCenter    = "production"
      BackupEnabled = "true"
      Criticality   = "high"
    }
    staging = {
      CostCenter = "non-production"
    }
    dev = {
      CostCenter  = "non-production"
      AutoShutdown = "true"
    }
  }

  service_tags = {
    Name    = "${var.service_name}-${var.environment}"
    Service = var.service_name
  }

  # Priority: base < env < service < custom_config (caller wins)
  final_tags = merge(
    local.base_tags,
    lookup(local.env_tags, var.environment, {}),
    local.service_tags,
    var.custom_config  # Highest priority
  )
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.latest.id
  instance_type = "t3.medium"
  tags          = local.final_tags
}
```

## Merging Provider Configurations

```hcl
# Reusable network configuration components
locals {
  # Base networking
  base_network = {
    assign_public_ip = false
    enable_ipv6      = false
  }

  # Public-facing services need public IPs
  public_network = {
    assign_public_ip = true
    security_groups  = ["sg-public"]
  }

  # Private services stay in private subnets
  private_network = {
    subnet_id       = aws_subnet.private.id
    security_groups = ["sg-private", "sg-internal"]
  }

  # Database network configuration
  db_network = merge(local.base_network, local.private_network, {
    security_groups = ["sg-private", "sg-db"]  # Override sg list
  })

  # Web tier configuration
  web_network = merge(local.base_network, local.public_network)

  # API tier (private but with specific security groups)
  api_network = merge(local.base_network, local.private_network, {
    security_groups = ["sg-private", "sg-api"]
  })
}
```

## Conditional Override Pattern

```hcl
variable "override_instance_type" {
  type    = string
  default = null
  description = "Set to override the default instance type (null uses environment default)"
}

variable "is_spot_instance" {
  type    = bool
  default = false
}

locals {
  base_config = {
    instance_type     = "t3.medium"
    capacity_type     = "ON_DEMAND"
    root_volume_size  = 30
  }

  # Only include override when non-null (empty map otherwise)
  instance_type_override = var.override_instance_type != null ? {
    instance_type = var.override_instance_type
  } : {}

  spot_override = var.is_spot_instance ? {
    capacity_type = "SPOT"
    spot_instance_interruption_behavior = "terminate"
  } : {}

  # Apply conditional overrides
  final_config = merge(
    local.base_config,
    local.instance_type_override,  # Applied only when override_instance_type is set
    local.spot_override,           # Applied only when is_spot_instance is true
  )
}

resource "aws_eks_node_group" "app" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "app-nodes"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = aws_subnet.private[*].id
  instance_types  = [local.final_config.instance_type]
  capacity_type   = local.final_config.capacity_type

  scaling_config {
    desired_size = 2
    min_size     = 1
    max_size     = 5
  }
}
```

## Summary

The `merge` function is the foundation of layered configuration management in OpenTofu. By ordering arguments from lowest to highest priority — global defaults, environment overrides, service overrides, caller-provided overrides — each layer can selectively override only the settings it cares about. The conditional override pattern using ternary expressions with empty maps (`var.x != null ? { key = var.x } : {}`) enables optional overrides without resorting to complex conditionals.
