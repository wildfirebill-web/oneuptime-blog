# How to Use the cidrcontains Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Networking

Description: Learn how to use the cidrcontains function in OpenTofu to check if an IP address or CIDR block falls within a given CIDR range.

## Introduction

The `cidrcontains` function in OpenTofu returns `true` if a given IP address or CIDR block is contained within a specified CIDR range. It is useful for network validation, security group rule verification, and IP address filtering in infrastructure configurations.

## Syntax

```hcl
cidrcontains(containing_cidr, contained_ip_or_cidr)
```

- **containing_cidr** - the outer CIDR range to check against
- **contained_ip_or_cidr** - an IP address or CIDR that may be inside it
- Returns a boolean

## Basic Examples

```hcl
output "ip_in_range" {
  value = cidrcontains("10.0.0.0/8", "10.5.3.1")       # Returns true
}

output "cidr_in_range" {
  value = cidrcontains("10.0.0.0/8", "10.1.0.0/16")    # Returns true
}

output "not_in_range" {
  value = cidrcontains("10.0.0.0/8", "192.168.1.1")    # Returns false
}
```

## Practical Use Cases

### Validating Security Group Rules

```hcl
variable "allowed_cidr" {
  type = string

  validation {
    condition     = cidrcontains("10.0.0.0/8", var.allowed_cidr)
    error_message = "allowed_cidr must be within the private 10.0.0.0/8 range."
  }
}
```

### Verifying Subnet is Within VPC

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_cidr" {
  type = string

  validation {
    condition     = cidrcontains(var.vpc_cidr, var.subnet_cidr)
    error_message = "subnet_cidr must be within the VPC CIDR ${var.vpc_cidr}."
  }
}
```

### Filtering IP Lists

```hcl
variable "ip_list" {
  type    = list(string)
  default = ["10.0.1.5", "192.168.1.1", "10.0.2.10", "172.16.0.1"]
}

locals {
  # Keep only IPs within 10.0.0.0/8
  private_ips = [
    for ip in var.ip_list :
    ip if cidrcontains("10.0.0.0/8", ip)
  ]
}

output "private_ips" {
  value = local.private_ips  # ["10.0.1.5", "10.0.2.10"]
}
```

### Detecting Internet-Accessible Ranges

```hcl
variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8", "0.0.0.0/0"]
}

locals {
  has_public_access = anytrue([
    for cidr in var.allowed_cidrs :
    cidrcontains(cidr, "0.0.0.0") || cidr == "0.0.0.0/0"
  ])
}

output "is_publicly_accessible" {
  value = local.has_public_access
}
```

## Step-by-Step Usage

```bash
tofu console

> cidrcontains("192.168.0.0/16", "192.168.1.100")
true
> cidrcontains("10.0.0.0/24", "10.0.1.0")
false
```

## Conclusion

The `cidrcontains` function is essential for network validation in OpenTofu. Use it in `validation` blocks to enforce network policies, in `for` expressions to filter IP addresses, and in `precondition` blocks to verify that subnets fall within their parent VPC CIDR.
