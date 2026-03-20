# How to Use the cidrsubnets Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Networking

Description: Learn how to use the cidrsubnets function in OpenTofu to allocate multiple subnets of different sizes from a parent CIDR block in a single call.

## Introduction

The `cidrsubnets` function in OpenTofu allocates multiple consecutive subnets from a parent CIDR block, where each subnet can have a different size specified by the number of new bits. Unlike `cidrsubnet`, which computes one subnet at a time, `cidrsubnets` computes a list of non-overlapping subnets in one call.

## Syntax

```hcl
cidrsubnets(prefix, newbits...)
```

- **prefix** - the parent CIDR block
- **newbits...** - one argument per desired subnet, each specifying additional prefix bits
- Returns a list of CIDR strings, one per requested subnet

## Basic Examples

```hcl
output "equal_subnets" {
  value = cidrsubnets("10.0.0.0/16", 8, 8, 8)
  # Returns ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}

output "mixed_size_subnets" {
  value = cidrsubnets("10.0.0.0/16", 4, 4, 8, 8)
  # Returns ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/24", "10.0.33.0/24"]
}
```

## Practical Use Cases

### Allocating Public and Private Subnets

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

locals {
  # Allocate 3 public /24 and 3 private /24 subnets
  all_subnets     = cidrsubnets(var.vpc_cidr, 8, 8, 8, 8, 8, 8)
  public_subnets  = slice(local.all_subnets, 0, 3)
  private_subnets = slice(local.all_subnets, 3, 6)
}

output "public_subnets" {
  value = local.public_subnets
  # ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}

output "private_subnets" {
  value = local.private_subnets
  # ["10.0.3.0/24", "10.0.4.0/24", "10.0.5.0/24"]
}
```

### Tiered Network with Different Sizes

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"

  # Allocate subnets of different sizes for different tiers
  network_plan = cidrsubnets(
    local.vpc_cidr,
    4,  # Public tier: 10.0.0.0/20 (4096 IPs)
    4,  # Private tier: 10.0.16.0/20 (4096 IPs)
    4,  # Database tier: 10.0.32.0/20 (4096 IPs)
    8,  # Management: 10.0.48.0/24 (256 IPs)
    8   # DMZ: 10.0.49.0/24 (256 IPs)
  )

  public_tier  = local.network_plan[0]
  private_tier = local.network_plan[1]
  db_tier      = local.network_plan[2]
  mgmt_subnet  = local.network_plan[3]
  dmz_subnet   = local.network_plan[4]
}
```

### Dynamic Subnet Count

```hcl
variable "az_count" {
  type    = number
  default = 3
}

locals {
  vpc_cidr = "10.0.0.0/16"
  # Create newbits list dynamically: one /24 per AZ for public, one for private
  newbits = [for i in range(var.az_count * 2) : 8]
  subnets = cidrsubnets(local.vpc_cidr, local.newbits...)
}
```

### Kubernetes Cluster Network

```hcl
locals {
  cluster_cidr = "10.0.0.0/14"

  # Node, pod, and service subnets
  k8s_subnets = cidrsubnets(
    local.cluster_cidr,
    2,   # Node subnet: 10.0.0.0/16
    2,   # Pod subnet: 10.1.0.0/16
    4    # Service subnet: 10.2.0.0/18
  )

  node_subnet    = local.k8s_subnets[0]
  pod_subnet     = local.k8s_subnets[1]
  service_subnet = local.k8s_subnets[2]
}
```

## Step-by-Step Usage

```bash
tofu console

> cidrsubnets("10.0.0.0/16", 8, 8, 8)
["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
> cidrsubnets("192.168.0.0/24", 2, 2)
["192.168.0.0/26", "192.168.0.64/26"]
```

## cidrsubnets vs cidrsubnet

| Function | Returns | Use Case |
|----------|---------|----------|
| `cidrsubnet(p, n, idx)` | Single CIDR | One subnet at a time |
| `cidrsubnets(p, n1, n2, ...)` | List of CIDRs | Multiple subnets in one call |

Use `cidrsubnets` when you know how many subnets you need upfront and want to specify different sizes.

## Conclusion

The `cidrsubnets` function is a powerful network planning tool in OpenTofu. It allocates multiple non-overlapping subnets from a parent CIDR in a single call, supporting different subnet sizes for different tiers. It is ideal for structured VPC designs where you need a predetermined set of subnets with specific size requirements.
