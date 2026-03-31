# How to Design a Load Balancer Module for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, ALB, AWS, Module, Load Balancing, Networking

Description: Learn how to design a reusable Application Load Balancer module for OpenTofu that supports HTTPS termination, multiple target groups, and routing rules.

## Introduction

A load balancer module should handle ALB creation, security groups, HTTPS listener setup, and target group creation in a single reusable unit. This makes it easy to deploy consistent load balancers across multiple services.

## variables.tf

```hcl
variable "name"           { type = string }
variable "environment"    { type = string }
variable "vpc_id"         { type = string }
variable "subnet_ids"     { type = list(string) }
variable "internal"       { type = bool; default = false }

variable "certificate_arn" {
  description = "ACM certificate ARN for HTTPS. Empty string = HTTP only."
  type        = string
  default     = ""
}

variable "target_groups" {
  type = map(object({
    port             = number
    protocol         = string
    health_check_path = string
    target_type      = string  # "instance" or "ip"
  }))
  default = {}
}

variable "allowed_ingress_cidrs" {
  type    = list(string)
  default = ["0.0.0.0/0"]
}

variable "idle_timeout"       { type = number; default = 60 }
variable "enable_deletion_protection" { type = bool; default = false }
variable "tags"               { type = map(string); default = {} }
```

## main.tf

```hcl
locals {
  tags = merge({ Environment = var.environment, ManagedBy = "OpenTofu" }, var.tags)
  enable_https = var.certificate_arn != ""
}

resource "aws_security_group" "alb" {
  name        = "${var.name}-alb-sg"
  description = "Security group for ${var.name} ALB"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.allowed_ingress_cidrs
    description = "HTTP"
  }

  dynamic "ingress" {
    for_each = local.enable_https ? [1] : []
    content {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = var.allowed_ingress_cidrs
      description = "HTTPS"
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.tags, { Name = "${var.name}-alb-sg" })
}

resource "aws_lb" "main" {
  name               = var.name
  internal           = var.internal
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.subnet_ids
  idle_timeout       = var.idle_timeout
  enable_deletion_protection = var.enable_deletion_protection
  tags               = merge(local.tags, { Name = var.name })
}

resource "aws_lb_target_group" "groups" {
  for_each = var.target_groups

  name        = "${var.name}-${each.key}"
  port        = each.value.port
  protocol    = each.value.protocol
  vpc_id      = var.vpc_id
  target_type = each.value.target_type

  health_check {
    path                = each.value.health_check_path
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    timeout             = 5
    matcher             = "200-299"
  }

  tags = merge(local.tags, { Name = "${var.name}-${each.key}" })
}

# HTTP listener - redirect to HTTPS if cert provided, forward otherwise

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  dynamic "default_action" {
    for_each = local.enable_https ? [1] : []
    content {
      type = "redirect"
      redirect {
        port        = "443"
        protocol    = "HTTPS"
        status_code = "HTTP_301"
      }
    }
  }

  dynamic "default_action" {
    for_each = local.enable_https ? [] : [1]
    content {
      type             = "fixed-response"
      fixed_response {
        content_type = "text/plain"
        message_body = "No target configured"
        status_code  = "503"
      }
    }
  }
}

# HTTPS listener (only when certificate is provided)
resource "aws_lb_listener" "https" {
  count             = local.enable_https ? 1 : 0
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "No target matched"
      status_code  = "404"
    }
  }
}
```

## outputs.tf

```hcl
output "alb_arn"           { value = aws_lb.main.arn }
output "alb_dns_name"      { value = aws_lb.main.dns_name }
output "alb_zone_id"       { value = aws_lb.main.zone_id }
output "security_group_id" { value = aws_security_group.alb.id }
output "http_listener_arn" { value = aws_lb_listener.http.arn }
output "https_listener_arn" {
  value = local.enable_https ? aws_lb_listener.https[0].arn : null
}
output "target_group_arns" {
  value = { for k, tg in aws_lb_target_group.groups : k => tg.arn }
}
```

## Conclusion

This ALB module handles the most common patterns: optional HTTPS with automatic HTTP-to-HTTPS redirect, multiple target groups for different services, and a managed security group. The conditional HTTPS listener pattern means the same module works for both HTTP-only (dev) and HTTPS (prod) deployments.
