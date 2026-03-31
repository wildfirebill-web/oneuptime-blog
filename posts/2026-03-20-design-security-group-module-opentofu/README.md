# How to Design a Security Group Module for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Security Group, AWS, Module, Networking, Security

Description: Learn how to design a reusable security group module for OpenTofu that accepts dynamic ingress and egress rules and supports both CIDR and security group source references.

## Introduction

A well-designed security group module abstracts away the verbose AWS security group rule syntax while remaining flexible enough to handle CIDR blocks, security group references, and both IPv4 and IPv6 sources.

## Module Structure

```text
modules/security-group/
├── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

## variables.tf

```hcl
variable "name"        { type = string }
variable "description" { type = string }
variable "vpc_id"      { type = string }
variable "environment" { type = string }

variable "ingress_rules" {
  description = "List of ingress rules"
  type = list(object({
    description              = string
    from_port                = number
    to_port                  = number
    protocol                 = string
    cidr_blocks              = optional(list(string), [])
    ipv6_cidr_blocks         = optional(list(string), [])
    source_security_group_id = optional(string)
    self                     = optional(bool, false)
  }))
  default = []
}

variable "egress_rules" {
  type = list(object({
    description      = string
    from_port        = number
    to_port          = number
    protocol         = string
    cidr_blocks      = optional(list(string), ["0.0.0.0/0"])
    ipv6_cidr_blocks = optional(list(string), [])
  }))
  default = [
    {
      description = "All outbound"
      from_port   = 0
      to_port     = 0
      protocol    = "-1"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
}

variable "tags" { type = map(string); default = {} }
```

## main.tf

```hcl
locals {
  tags = merge({
    Name        = var.name
    Environment = var.environment
    ManagedBy   = "OpenTofu"
  }, var.tags)
}

resource "aws_security_group" "main" {
  name        = var.name
  description = var.description
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      description              = ingress.value.description
      from_port                = ingress.value.from_port
      to_port                  = ingress.value.to_port
      protocol                 = ingress.value.protocol
      cidr_blocks              = length(ingress.value.cidr_blocks) > 0 ? ingress.value.cidr_blocks : null
      ipv6_cidr_blocks         = length(ingress.value.ipv6_cidr_blocks) > 0 ? ingress.value.ipv6_cidr_blocks : null
      source_security_group_id = ingress.value.source_security_group_id
      self                     = ingress.value.self
    }
  }

  dynamic "egress" {
    for_each = var.egress_rules
    content {
      description      = egress.value.description
      from_port        = egress.value.from_port
      to_port          = egress.value.to_port
      protocol         = egress.value.protocol
      cidr_blocks      = length(egress.value.cidr_blocks) > 0 ? egress.value.cidr_blocks : null
      ipv6_cidr_blocks = length(egress.value.ipv6_cidr_blocks) > 0 ? egress.value.ipv6_cidr_blocks : null
    }
  }

  tags = local.tags

  lifecycle {
    create_before_destroy = true
  }
}
```

## Example Usage

```hcl
module "app_sg" {
  source      = "./modules/security-group"
  name        = "app-server-sg"
  description = "Security group for application servers"
  vpc_id      = module.vpc.vpc_id
  environment = var.environment

  ingress_rules = [
    {
      description = "HTTPS from internet"
      from_port   = 443; to_port = 443; protocol = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description              = "App port from ALB"
      from_port                = 8080; to_port = 8080; protocol = "tcp"
      source_security_group_id = module.alb_sg.security_group_id
    }
  ]
}
```

## outputs.tf

```hcl
output "security_group_id"  { value = aws_security_group.main.id }
output "security_group_arn" { value = aws_security_group.main.arn }
output "security_group_name" { value = aws_security_group.main.name }
```

## Conclusion

This security group module uses dynamic blocks to handle any number of rules, and the `optional()` type constraints provide sensible defaults so callers only specify what they need. The `create_before_destroy` lifecycle rule prevents service interruptions when security groups need to be replaced.
