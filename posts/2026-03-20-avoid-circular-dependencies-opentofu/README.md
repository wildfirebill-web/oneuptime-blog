# How to Avoid Circular Dependencies in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Dependencies, Best Practice, Troubleshooting, Infrastructure as Code

Description: Learn how to identify, diagnose, and fix circular dependencies in OpenTofu configurations.

## Introduction

A circular dependency occurs when Resource A depends on Resource B, and Resource B depends on Resource A. OpenTofu cannot determine which to create first and will error with "Cycle detected." Understanding how to restructure configurations to eliminate cycles is a practical skill for infrastructure engineers.

## How Circular Dependencies Occur

The most common pattern is mutual security group references.

```hcl
# This creates a cycle: sg_a → sg_b → sg_a

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.load_balancer.id]  # depends on lb_sg
  }
}

resource "aws_security_group" "load_balancer" {
  name   = "lb-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # depends on app_sg → CYCLE
  }
}
```

## Diagnosing the Cycle

```bash
# OpenTofu will show the cycle when you run plan
tofu plan

# Error output:
# Error: Cycle detected
#
# module.root:
#   aws_security_group.app
#   aws_security_group.load_balancer

# Also use the graph command to visualize
tofu graph | grep -A5 -B5 "security_group"
```

## Fix: Use Separate Security Group Rules

Break the cycle by separating security group creation from rule attachment.

```hcl
# Create security groups without inline rules
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
  # No inline ingress/egress rules
}

resource "aws_security_group" "load_balancer" {
  name   = "lb-sg"
  vpc_id = aws_vpc.main.id
  # No inline ingress/egress rules
}

# Add rules separately - no cycle because both SGs exist first
resource "aws_security_group_rule" "app_from_lb" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  security_group_id        = aws_security_group.app.id
  source_security_group_id = aws_security_group.load_balancer.id
}

resource "aws_security_group_rule" "lb_to_app" {
  type                     = "egress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  security_group_id        = aws_security_group.load_balancer.id
  source_security_group_id = aws_security_group.app.id
}
```

## Another Common Cycle: Module Outputs

Cycles can occur between modules when they reference each other's outputs.

```hcl
# BAD: Module A uses module B's output, Module B uses Module A's output
module "app" {
  source         = "./modules/app"
  security_group = module.network.app_security_group_id
}

module "network" {
  source        = "./modules/network"
  app_subnet_id = module.app.subnet_id  # CYCLE
}

# FIX: Restructure so one module doesn't depend on the other
module "network" {
  source = "./modules/network"
  # network has no dependency on app module
}

module "app" {
  source         = "./modules/app"
  subnet_id      = module.network.private_subnet_id
  security_group = module.network.app_security_group_id
}
```

## Avoiding depends_on Cycles

Explicit `depends_on` can also create cycles.

```hcl
# BAD: Explicit cycle via depends_on
resource "aws_iam_role" "lambda" {
  name       = "lambda-role"
  depends_on = [aws_lambda_function.processor]  # depends on lambda
}

resource "aws_lambda_function" "processor" {
  role       = aws_iam_role.lambda.arn
  depends_on = [aws_iam_role.lambda]  # depends on role (already implicit)
}

# FIX: Remove the unnecessary depends_on from the role
# The implicit dependency through aws_lambda_function.processor.role = aws_iam_role.lambda.arn
# is sufficient and correct
resource "aws_iam_role" "lambda" {
  name = "lambda-role"
  # No depends_on needed here
}
```

## Summary

Circular dependencies in OpenTofu are resolved by restructuring how resources reference each other. The most common fix is separating resource creation from relationship configuration - create security groups first, then add rules separately. For module cycles, ensure data flows in one direction (network module → app module, never the reverse). Use `tofu graph` to visualize dependencies when diagnosing complex cycles. Avoid adding `depends_on` to resources that already have implicit dependencies through attribute references.
