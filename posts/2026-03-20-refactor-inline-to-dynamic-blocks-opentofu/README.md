# How to Refactor Inline Blocks to Dynamic Blocks in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Dynamic Blocks, Refactoring, HCL, Infrastructure as Code

Description: Learn how to refactor repetitive inline resource blocks in OpenTofu to dynamic blocks for cleaner, more maintainable configurations.

When the same type of nested block appears multiple times within a resource - security group rules, listener rules, environment variables - inline repetition becomes hard to maintain. Dynamic blocks replace repeated inline blocks with a single, loop-driven block that generates them from a variable or local.

## The Problem: Repetitive Inline Blocks

```hcl
# Hard to maintain: adding a rule requires editing the resource directly

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = [var.internal_cidr]
  }
}
```

## The Solution: Dynamic Block

```hcl
# Clean: rules are a variable, resource block doesn't change
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { from_port = 80,   to_port = 80,   protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { from_port = 443,  to_port = 443,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { from_port = 8080, to_port = 8080, protocol = "tcp", cidr_blocks = ["10.0.0.0/8"] },
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  # Generate one ingress block per rule in the variable
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

## Dynamic ECS Container Definitions

Another common use case - ECS task definition environment variables:

```hcl
variable "env_vars" {
  type    = map(string)
  default = { LOG_LEVEL = "info", PORT = "8080" }
}

resource "aws_ecs_task_definition" "app" {
  family = "my-app"

  container_definitions = jsonencode([{
    name  = "app"
    image = var.image_uri

    # Generate environment block for each key-value pair
    environment = [
      for k, v in var.env_vars : { name = k, value = v }
    ]
  }])
}
```

## Dynamic ALB Listener Rules

```hcl
variable "path_routing_rules" {
  type = list(object({
    path            = string
    target_group_arn = string
    priority        = number
  }))
}

resource "aws_lb_listener_rule" "path_rules" {
  for_each     = { for r in var.path_routing_rules : r.path => r }
  listener_arn = aws_lb_listener.http.arn
  priority     = each.value.priority

  action {
    type             = "forward"
    target_group_arn = each.value.target_group_arn
  }

  condition {
    path_pattern { values = [each.value.path] }
  }
}
```

## Conditional Dynamic Blocks

Use `for_each` with an empty list to optionally include a block:

```hcl
variable "enable_access_logs" {
  type    = bool
  default = false
}

resource "aws_lb" "main" {
  name               = "main-lb"
  load_balancer_type = "application"

  # access_logs block only added when enabled
  dynamic "access_logs" {
    for_each = var.enable_access_logs ? [1] : []
    content {
      bucket  = var.log_bucket
      enabled = true
    }
  }
}
```

## Conclusion

Dynamic blocks eliminate repetitive inline block definitions, making resources configurable through variables rather than requiring edits to resource blocks. Refactor inline blocks to dynamic blocks whenever a resource has more than two or three instances of the same block type, and move the block data into a typed variable for clean module interfaces.
