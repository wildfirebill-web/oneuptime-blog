# How to Validate IP Addresses with cidrcontains in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, CIDR, Networking, Validation, Functions

Description: Learn how to use the cidrcontains function in OpenTofu to validate whether IP addresses fall within CIDR ranges, enforce network policies, and prevent misconfigured security rules.

## Overview

The `cidrcontains` function in OpenTofu checks whether a given IP address or CIDR block falls within a larger CIDR range. It is particularly useful for validating network configurations, enforcing subnet policies, and ensuring security group rules conform to expected address spaces before applying infrastructure changes.

## Basic cidrcontains Usage

```hcl
# Check if an IP address falls within a CIDR block

locals {
  vpc_cidr     = "10.0.0.0/16"
  db_server_ip = "10.0.5.42"
  external_ip  = "192.168.1.1"

  db_in_vpc  = cidrcontains(local.vpc_cidr, local.db_server_ip)  # true
  ext_in_vpc = cidrcontains(local.vpc_cidr, local.external_ip)   # false
}

output "db_server_in_vpc" {
  value = local.db_in_vpc  # true
}

output "external_ip_in_vpc" {
  value = local.ext_in_vpc  # false
}
```

## Input Validation with cidrcontains

```hcl
variable "allowed_vpc_cidr" {
  type        = string
  default     = "10.0.0.0/8"
  description = "The allowed VPC CIDR range for private IPs"
}

variable "vpc_cidr" {
  type        = string
  description = "The CIDR block for the VPC"

  validation {
    # Ensure the VPC CIDR is within the allowed private range
    condition     = cidrcontains(var.allowed_vpc_cidr, var.vpc_cidr)
    error_message = "VPC CIDR must be within the allowed private range ${var.allowed_vpc_cidr}."
  }
}

variable "nat_gateway_ip" {
  type        = string
  description = "The EIP to assign to the NAT Gateway (must be outside the VPC CIDR)"

  validation {
    # NAT gateway IP must NOT be within the VPC CIDR
    condition     = !cidrcontains(var.vpc_cidr, var.nat_gateway_ip)
    error_message = "NAT Gateway EIP must be a public IP, not within VPC CIDR ${var.vpc_cidr}."
  }
}
```

## Security Group Validation

```hcl
variable "allowed_ingress_cidrs" {
  type        = list(string)
  description = "CIDRs allowed to access the application"
  default     = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
}

variable "additional_cidr" {
  type        = string
  description = "Additional CIDR to allow - must be private"

  validation {
    # Only allow private RFC 1918 address space
    condition = anytrue([
      cidrcontains("10.0.0.0/8", var.additional_cidr),
      cidrcontains("172.16.0.0/12", var.additional_cidr),
      cidrcontains("192.168.0.0/16", var.additional_cidr),
    ])
    error_message = "Only private RFC 1918 address space is allowed. Received: ${var.additional_cidr}."
  }
}

resource "aws_security_group_rule" "additional_ingress" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = [var.additional_cidr]
  security_group_id = aws_security_group.app.id
}
```

## Subnet-to-VPC Membership Validation

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.100.0.0/16"
}

variable "subnet_cidrs" {
  type = map(string)
  default = {
    public_a  = "10.100.1.0/24"
    public_b  = "10.100.2.0/24"
    private_a = "10.100.10.0/24"
    private_b = "10.100.11.0/24"
  }
}

locals {
  # Validate all subnets belong to the VPC CIDR
  subnet_validation = {
    for name, cidr in var.subnet_cidrs :
    name => cidrcontains(var.vpc_cidr, cidr)
  }

  # Collect invalid subnets for reporting
  invalid_subnets = [
    for name, valid in local.subnet_validation :
    name if !valid
  ]
}

# Fail early if any subnet is outside the VPC CIDR
resource "terraform_data" "subnet_validation" {
  lifecycle {
    precondition {
      condition     = length(local.invalid_subnets) == 0
      error_message = "The following subnets are outside the VPC CIDR ${var.vpc_cidr}: ${join(", ", local.invalid_subnets)}"
    }
  }
}

resource "aws_subnet" "subnets" {
  for_each          = var.subnet_cidrs
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  availability_zone = "us-east-1a"

  tags = {
    Name = each.key
  }
}
```

## Firewall Rule Compliance Checking

```hcl
locals {
  # Define trusted network zones
  trusted_zones = {
    corporate_vpn  = "172.31.0.0/16"
    office_network = "192.168.0.0/24"
    internal_vpc   = "10.0.0.0/8"
  }

  # Proposed firewall rules from variable or data source
  proposed_rules = [
    { name = "vpn-access",      cidr = "172.31.5.0/24" },
    { name = "office-access",   cidr = "192.168.0.100/32" },
    { name = "external-access", cidr = "203.0.113.0/24" },  # Not in any trusted zone
  ]

  # Flag rules that are outside all trusted zones
  untrusted_rules = [
    for rule in local.proposed_rules :
    rule.name if !anytrue([
      for zone_cidr in values(local.trusted_zones) :
      cidrcontains(zone_cidr, rule.cidr)
    ])
  ]
}

output "untrusted_firewall_rules" {
  description = "Rules referencing IPs outside trusted zones (require approval)"
  value       = local.untrusted_rules
}

output "all_rules_trusted" {
  value = length(local.untrusted_rules) == 0
}
```

## Conditional Resource Creation Based on IP Range

```hcl
variable "peer_vpc_cidr" {
  type = string
}

locals {
  # Determine if the peer is in the same /8 supernet (same org)
  same_org_network = cidrcontains("10.0.0.0/8", var.peer_vpc_cidr)

  # Use different peering configs based on network ownership
  peering_tags = local.same_org_network ? {
    PeeringType = "internal"
    AutoApprove = "true"
  } : {
    PeeringType = "external"
    AutoApprove = "false"
  }
}

resource "aws_vpc_peering_connection" "peer" {
  vpc_id      = aws_vpc.main.id
  peer_vpc_id = aws_vpc.peer.id
  auto_accept = local.same_org_network  # Auto-accept only for internal peerings

  tags = merge(local.peering_tags, {
    Name = "vpc-peer-${var.peer_vpc_cidr}"
  })
}
```

## Summary

The `cidrcontains` function enables network policy enforcement directly within OpenTofu configurations. By combining it with `validation` blocks, `precondition` lifecycle rules, and `anytrue` aggregation, you can catch misconfigured IP ranges before resources are created. This prevents costly security mistakes like accidentally opening security groups to public internet addresses when only private access was intended.
