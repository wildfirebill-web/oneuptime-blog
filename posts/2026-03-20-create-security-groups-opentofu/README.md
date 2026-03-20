# How to Create Security Groups with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, AWS, Security Groups, Terraform, IaC, DevOps

Description: Learn how to create AWS security groups with OpenTofu using both inline rules and separate rule resources for scalable network access control.

## Introduction

Security groups are virtual firewalls for your AWS resources. OpenTofu supports two approaches for managing security group rules: inline `ingress`/`egress` blocks within the `aws_security_group` resource, or separate `aws_security_group_rule` resources. Understanding when to use each approach is key to avoiding rule conflicts.

## Approach 1: Inline Rules (Simpler)

Best for security groups with a fixed, known set of rules:

```hcl
resource "aws_security_group" "web" {
  name        = "${var.environment}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from internet (for redirect)"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.environment}-web-sg" }
}
```

## Approach 2: Separate Rule Resources (More Flexible)

Best for security groups that receive rules from multiple modules or dynamically:

```hcl
# Create the security group without inline rules
resource "aws_security_group" "app" {
  name        = "${var.environment}-app-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  # Explicitly empty — rules added separately
  lifecycle {
    ignore_changes = [ingress, egress]  # Prevents conflicts with separate rules
  }

  tags = { Name = "${var.environment}-app-sg" }
}

# Add rules separately
resource "aws_security_group_rule" "app_http_in" {
  type              = "ingress"
  security_group_id = aws_security_group.app.id
  description       = "HTTP from ALB"
  from_port         = 8080
  to_port           = 8080
  protocol          = "tcp"
  source_security_group_id = aws_security_group.alb.id
}

resource "aws_security_group_rule" "app_all_out" {
  type              = "egress"
  security_group_id = aws_security_group.app.id
  description       = "All outbound"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
}
```

## Security Group with Self Reference

Allow instances in the same security group to communicate:

```hcl
resource "aws_security_group" "cluster" {
  name   = "${var.environment}-cluster-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description = "Inter-node communication"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    self        = true  # Allow traffic from other instances in this SG
  }
}
```

## Tiered Security Groups Pattern

A common pattern for web → app → db tiers:

```hcl
# Load balancer — accepts internet traffic
resource "aws_security_group" "alb" {
  name   = "${var.environment}-alb-sg"
  vpc_id = aws_vpc.main.id

  ingress { from_port = 443; to_port = 443; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"]; description = "HTTPS" }
  ingress { from_port = 80; to_port = 80; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"]; description = "HTTP" }
  egress  { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}

# App tier — accepts traffic from ALB only
resource "aws_security_group" "app" {
  name   = "${var.environment}-app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "From ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  egress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}

# Database tier — accepts traffic from app only
resource "aws_security_group" "db" {
  name   = "${var.environment}-db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL from app"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
  egress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}
```

## Dynamic Rules from Variables

```hcl
variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

resource "aws_security_group" "admin" {
  name   = "${var.environment}-admin-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.allowed_cidrs
    content {
      description = "SSH from ${ingress.value}"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
    }
  }
}
```

## Outputs

```hcl
output "alb_sg_id" { value = aws_security_group.alb.id }
output "app_sg_id" { value = aws_security_group.app.id }
output "db_sg_id"  { value = aws_security_group.db.id }
```

## Conclusion

Choose inline rules for simple, fixed security groups and separate `aws_security_group_rule` resources for complex groups that receive rules from multiple sources. The tiered security group pattern (ALB → App → DB) enforces defense-in-depth. Use security group references (`source_security_group_id`) instead of CIDR blocks for internal traffic — this ensures rules stay accurate even when instances change IP addresses.
