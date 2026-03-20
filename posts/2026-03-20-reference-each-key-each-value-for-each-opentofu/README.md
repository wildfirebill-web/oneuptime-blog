# How to Reference each.key and each.value in for_each in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, for_each, each.key, each.value, HCL

Description: Learn how to use each.key and each.value inside for_each resource blocks in OpenTofu to access the current iteration's identifier and configuration data.

## Introduction

Inside a `for_each` resource block, `each.key` is the map key (or set element) and `each.value` is the associated value. Together they give you full access to both the identifier and configuration of the current iteration, enabling rich per-instance customization.

## each.key and each.value with a Map

```hcl
variable "security_groups" {
  type = map(object({
    description  = string
    ingress_cidr = string
    ports        = list(number)
  }))
  default = {
    "web"      = { description = "Web tier SG",      ingress_cidr = "0.0.0.0/0",   ports = [80, 443] }
    "app"      = { description = "App tier SG",      ingress_cidr = "10.0.1.0/24", ports = [8080] }
    "database" = { description = "Database tier SG", ingress_cidr = "10.0.2.0/24", ports = [5432] }
  }
}

resource "aws_security_group" "tiers" {
  for_each = var.security_groups

  # each.key is "web", "app", or "database"
  name        = "${each.key}-security-group"
  description = each.value.description
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = each.value.ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      # each.value is still accessible inside nested dynamic blocks
      cidr_blocks = [each.value.ingress_cidr]
      description = "Port ${ingress.value} from ${each.key} tier"
    }
  }

  tags = {
    Name = "${each.key}-sg"
    Tier = each.key
  }
}
```

## Using each.key in Resource References

`each.key` can reference the same or another `for_each` resource by key.

```hcl
resource "aws_lb_target_group" "services" {
  for_each = var.services

  name     = "tg-${each.key}"
  port     = each.value.port
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener_rule" "services" {
  for_each = var.services

  listener_arn = aws_lb_listener.https.arn
  priority     = each.value.listener_priority

  action {
    type             = "forward"
    # Use each.key to reference the corresponding target group
    target_group_arn = aws_lb_target_group.services[each.key].arn
  }

  condition {
    path_pattern {
      values = each.value.path_patterns
    }
  }
}
```

## each.key as a Naming Component

```hcl
variable "s3_buckets" {
  type = map(object({
    versioning = bool
    lifecycle_days = number
  }))
}

data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "buckets" {
  for_each = var.s3_buckets

  # Incorporate each.key into bucket name and tags
  bucket = "${var.project}-${each.key}-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name    = each.key
    Project = var.project
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "buckets" {
  for_each = {
    for name, config in var.s3_buckets : name => config
    if config.lifecycle_days > 0
  }

  # Reference the parent bucket using each.key
  bucket = aws_s3_bucket.buckets[each.key].id

  rule {
    id     = "expire-objects"
    status = "Enabled"
    expiration {
      days = each.value.lifecycle_days
    }
  }
}
```

## each.value with Nested Object Access

```hcl
variable "clusters" {
  type = map(object({
    node_groups = map(object({
      instance_type = string
      min_size      = number
      max_size      = number
    }))
  }))
}

# Flatten to one resource per cluster-nodegroup pair
locals {
  node_groups = {
    for item in flatten([
      for cluster_name, cluster in var.clusters : [
        for ng_name, ng in cluster.node_groups : {
          key          = "${cluster_name}-${ng_name}"
          cluster_name = cluster_name
          ng_name      = ng_name
          config       = ng
        }
      ]
    ]) : item.key => item
  }
}

resource "aws_eks_node_group" "groups" {
  for_each = local.node_groups

  cluster_name    = each.value.cluster_name
  node_group_name = each.value.ng_name
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.subnet_ids

  instance_types = [each.value.config.instance_type]

  scaling_config {
    min_size = each.value.config.min_size
    max_size = each.value.config.max_size
    desired_size = each.value.config.min_size
  }
}
```

## Conclusion

`each.key` and `each.value` are the primary tools for customizing resources within a `for_each` block. Use `each.key` for naming and cross-referencing, `each.value` (with nested property access using `.`) for configuration values. Both are accessible in dynamic sub-blocks within the same resource, making deeply customized per-instance configurations straightforward.
