# How to Configure Security Groups for IPv4 Using Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Terraform, AWS, Security Groups, IPv4, Infrastructure as Code, Networking

Description: Configure AWS security groups for IPv4 using Terraform, covering ingress/egress rules, CIDR-based and security-group-based references, and best practices for least privilege.

## Introduction

AWS security groups act as stateful firewalls for EC2 instances and other resources. Terraform's `aws_security_group` and `aws_security_group_rule` resources manage them declaratively.

## Web Server Security Group

```hcl
# security_groups.tf

resource "aws_security_group" "web" {
  name        = "web-servers"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  # HTTP inbound
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow HTTP from internet"
  }

  # HTTPS inbound
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow HTTPS from internet"
  }

  # SSH only from management subnet
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.64.99.0/27"]
    description = "SSH from management subnet"
  }

  # All outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name = "web-sg"
  }
}
```

## Database Security Group (Reference Another SG)

```hcl
resource "aws_security_group" "database" {
  name        = "database"
  description = "RDS database security group"
  vpc_id      = aws_vpc.main.id

  # Allow only from web tier SG
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
    description     = "PostgreSQL from web tier"
  }

  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.64.99.0/27"]  # Management subnet
    description = "PostgreSQL from management"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "database-sg" }
}
```

## Separate Rule Resources (Avoid Circular Dependencies)

```hcl
resource "aws_security_group" "app" {
  name   = "app-servers"
  vpc_id = aws_vpc.main.id
}

resource "aws_security_group_rule" "app_from_web" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.web.id
  security_group_id        = aws_security_group.app.id
  description              = "App port from web tier"
}
```

## Outputs

```hcl
output "web_sg_id" {
  value = aws_security_group.web.id
}

output "database_sg_id" {
  value = aws_security_group.database.id
}
```

## Deploy

```bash
terraform plan
terraform apply
terraform output web_sg_id
```

## Conclusion

AWS security groups in Terraform use `ingress`/`egress` blocks for inline rules or separate `aws_security_group_rule` resources for modular management. Reference other security groups with `security_groups = [sg_id]` to create dynamic rules that follow instance membership changes. Restrict SSH and admin ports to management CIDR ranges rather than `0.0.0.0/0`.
