# How to Create Security Groups with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Security Group, Networking, Infrastructure as Code

Description: Learn how to create and manage AWS security groups with OpenTofu, including inbound/outbound rules, security group references, and lifecycle management.

## Introduction

Security groups act as virtual firewalls for EC2 instances, RDS databases, ELBs, and other AWS resources. OpenTofu provides two approaches: inline `ingress`/`egress` blocks within `aws_security_group`, or separate `aws_security_group_rule` resources for more flexible management.

## Basic Security Group

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  # Allow HTTP from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from internet"
  }

  # Allow HTTPS from anywhere
  ingress {
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
    description      = "HTTPS from internet"
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}
```

## Referencing Security Groups (Source Security Group)

Allow traffic between security groups without hardcoding CIDRs:

```hcl
# Application layer security group

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id

  # Only accept traffic from the web (ALB) security group
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "App traffic from ALB only"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Database security group that only accepts connections from the app layer
resource "aws_security_group" "rds" {
  name   = "rds-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
    description     = "PostgreSQL from app layer only"
  }
}
```

## Using Separate Rule Resources

Separate `aws_security_group_rule` resources allow modular rule management:

```hcl
# Create the security group with no inline rules
resource "aws_security_group" "base" {
  name        = "base-sg"
  description = "Base security group - rules managed separately"
  vpc_id      = aws_vpc.main.id

  # Prevent Terraform from managing ingress/egress inline
  lifecycle {
    ignore_changes = [ingress, egress]
  }
}

# Add rules as separate resources
resource "aws_security_group_rule" "ssh_ingress" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = [var.bastion_cidr]
  security_group_id = aws_security_group.base.id
  description       = "SSH from bastion"
}

resource "aws_security_group_rule" "all_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.base.id
}
```

## Dynamic Rules from a Variable

```hcl
variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 8080]
}

resource "aws_security_group" "dynamic" {
  name   = "dynamic-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.allowed_ports

    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Conclusion

Security groups in OpenTofu are most maintainable when you use security group references instead of CIDRs for inter-tier traffic. Use inline rules for simple cases and separate `aws_security_group_rule` resources when multiple modules need to add rules to a shared security group.
