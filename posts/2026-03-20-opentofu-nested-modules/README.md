# Understanding Nested Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Module, Nested

Description: Learn how nested modules work in OpenTofu, when to use them, and how to structure multi-level module hierarchies.

Nested modules - modules that call other modules - allow you to build infrastructure abstractions at multiple levels. A high-level "application" module might call a "network" module, a "compute" module, and a "database" module, each of which may call even lower-level modules.

## Basic Nested Module Structure

```text
infrastructure/
├── main.tf              # Root module
├── variables.tf
├── outputs.tf
└── modules/
    ├── application/     # High-level module
    │   ├── main.tf
    │   ├── variables.tf
    │   ├── outputs.tf
    │   └── modules/     # Nested within application
    │       ├── web/
    │       └── api/
    └── networking/      # Another top-level module
        ├── main.tf
        └── modules/
            ├── vpc/
            └── subnets/
```

## Creating a Nested Module Hierarchy

```hcl
# modules/networking/main.tf - calls sub-modules

module "vpc" {
  source = "./modules/vpc"

  cidr_block  = var.vpc_cidr
  environment = var.environment
}

module "subnets" {
  source = "./modules/subnets"

  vpc_id      = module.vpc.vpc_id
  vpc_cidr    = var.vpc_cidr
  az_count    = var.availability_zone_count
  environment = var.environment
}

module "nat_gateway" {
  source = "./modules/nat"

  public_subnet_ids = module.subnets.public_subnet_ids
  environment       = var.environment
}
```

```hcl
# modules/networking/outputs.tf
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "private_subnet_ids" {
  value = module.subnets.private_subnet_ids
}

output "public_subnet_ids" {
  value = module.subnets.public_subnet_ids
}
```

```hcl
# Root main.tf - uses the high-level networking module
module "networking" {
  source = "./modules/networking"

  vpc_cidr                = "10.0.0.0/16"
  availability_zone_count = 3
  environment             = "prod"
}

module "application" {
  source = "./modules/application"

  vpc_id             = module.networking.vpc_id
  private_subnet_ids = module.networking.private_subnet_ids
  environment        = "prod"
}
```

## Passing Data Between Nested Levels

```hcl
# modules/application/main.tf
variable "vpc_id" {
  description = "VPC ID from the networking module"
  type        = string
}

variable "private_subnet_ids" {
  description = "List of private subnet IDs"
  type        = list(string)
}

# Application module creates its own sub-resources
module "web_tier" {
  source = "./modules/web"

  vpc_id     = var.vpc_id            # Pass through from parent
  subnet_ids = var.private_subnet_ids
  
  instance_count = var.web_instance_count
  instance_type  = var.instance_type
}

module "api_tier" {
  source = "./modules/api"

  vpc_id          = var.vpc_id
  subnet_ids      = var.private_subnet_ids
  web_sg_id       = module.web_tier.security_group_id  # Cross-module reference
  instance_count  = var.api_instance_count
}
```

## Referencing Nested Module Outputs

```hcl
# Outputs from nested modules bubble up through the hierarchy
# modules/application/outputs.tf
output "web_url" {
  description = "URL of the web tier load balancer"
  value       = module.web_tier.load_balancer_dns
}

output "api_url" {
  description = "URL of the API tier"
  value       = module.api_tier.endpoint
}

# Root outputs.tf - reference the application module's output
output "application_url" {
  description = "Application URL"
  value       = module.application.web_url
}
```

## When to Use Nested Modules

Good use cases for nesting:

```hcl
# Pattern 1: Large module decomposed into sub-modules
# modules/eks-cluster/main.tf
module "control_plane" {
  source         = "./modules/control-plane"
  cluster_name   = var.cluster_name
  k8s_version    = var.kubernetes_version
  vpc_id         = var.vpc_id
  subnet_ids     = var.subnet_ids
}

module "node_groups" {
  source         = "./modules/node-groups"
  cluster_name   = module.control_plane.cluster_name
  node_configs   = var.node_group_configs
  subnet_ids     = var.subnet_ids
}

module "addons" {
  source         = "./modules/addons"
  cluster_name   = module.control_plane.cluster_name
  oidc_issuer    = module.control_plane.oidc_issuer
  enabled_addons = var.enabled_addons
}
```

## When NOT to Use Deeply Nested Modules

Avoid more than 2-3 levels of nesting:

```hcl
# TOO DEEP - hard to understand and debug
module "root" {
  source = "./modules/level1"
}
# level1 calls level2 which calls level3 which calls level4
# When something breaks, tracing the issue is painful

# BETTER - keep it flat where possible
module "vpc" {
  source = "./modules/vpc"
}

module "eks" {
  source = "./modules/eks"
  vpc_id = module.vpc.vpc_id
}

module "rds" {
  source = "./modules/rds"
  vpc_id = module.vpc.vpc_id
}
```

## Conclusion

Nested modules are powerful for organizing large infrastructure codebases, but depth should be kept to a minimum. Use nesting to decompose complex modules into manageable pieces, but keep the overall hierarchy shallow enough that developers can understand the full picture without tracing through multiple levels.
