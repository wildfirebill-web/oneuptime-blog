# How to Use count with Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn how to use the count meta-argument with OpenTofu modules to conditionally create or replicate module instances.

## Introduction

The `count` meta-argument can be applied to module blocks in OpenTofu, just like resources. It enables conditional module creation (count = 0 or 1) and module replication (count = n). Each instance becomes addressable as `module.name[index]`.

## Syntax

```hcl
module "example" {
  source = "./modules/my-module"
  count  = var.enable_feature ? 1 : 0

  name = "example-${count.index}"
}
```

## Practical Use Cases

### Conditional Module Creation

```hcl
variable "enable_monitoring" {
  type    = bool
  default = true
}

module "monitoring_stack" {
  source = "./modules/monitoring"
  count  = var.enable_monitoring ? 1 : 0

  environment = var.environment
  cluster_id  = module.eks.cluster_id
}

output "monitoring_endpoint" {
  value = var.enable_monitoring ? module.monitoring_stack[0].grafana_url : null
}
```

### Conditional Bastion Host

```hcl
variable "create_bastion" {
  type    = bool
  default = false
}

module "bastion" {
  source = "./modules/bastion"
  count  = var.create_bastion ? 1 : 0

  subnet_id         = module.vpc.public_subnet_ids[0]
  security_group_id = aws_security_group.bastion.id
}

output "bastion_ip" {
  value = var.create_bastion ? one(module.bastion[*].public_ip) : null
}
```

### Multiple Instances

```hcl
variable "worker_count" {
  type    = number
  default = 3
}

module "worker" {
  source = "./modules/worker-node"
  count  = var.worker_count

  name              = "worker-${count.index + 1}"
  subnet_id         = element(module.vpc.private_subnet_ids, count.index)
  instance_type     = "t3.medium"
  environment       = var.environment
}

output "worker_ips" {
  value = [for w in module.worker : w.private_ip]
}
```

### Replicating Across Environments

```hcl
variable "environments" {
  type    = list(string)
  default = ["dev", "staging", "prod"]
}

module "app_stack" {
  source = "./modules/application"
  count  = length(var.environments)

  environment   = var.environments[count.index]
  instance_type = var.environments[count.index] == "prod" ? "m5.large" : "t3.small"
}
```

## Accessing Module Outputs with count

```hcl
# Single output from all instances

output "all_instance_ids" {
  value = [for m in module.worker : m.instance_id]
}

# Splat operator
output "all_private_ips" {
  value = module.worker[*].private_ip
}

# Specific instance
output "first_worker_ip" {
  value = module.worker[0].private_ip
}

# Optional instance (when count = 0 or 1)
output "bastion_ip" {
  value = one(module.bastion[*].public_ip)
}
```

## count vs for_each for Modules

| | `count` | `for_each` |
|-|---------|------------|
| Index type | Integer | String key |
| Adding items | May change all | Only adds new |
| Removing items | May destroy all | Only removes specific |
| Recommendation | Simple toggle | Named instances |

Use `for_each` when instances have meaningful names; use `count` for simple on/off or numbered replicas.

## Step-by-Step Usage

1. Add `count = expression` to the module block.
2. Use `count.index` for differentiation within the module.
3. Access outputs with `module.name[*].output` or `module.name[index].output`.

## Conclusion

The `count` meta-argument enables conditional and replicated module instantiation in OpenTofu. Use it for feature flags (`count = 0 or 1`) and simple replication (`count = n`). For instances that need stable identities, prefer `for_each` instead.
