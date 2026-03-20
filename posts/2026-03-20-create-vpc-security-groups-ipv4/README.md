# How to Create VPC Security Groups for IPv4 with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPC, Security Groups, IPv4, Networking

Description: Learn how to create AWS VPC security groups with IPv4 ingress and egress rules using OpenTofu for fine-grained network access control.

---

Security groups act as virtual firewalls for EC2 instances, RDS databases, and other AWS resources. They are stateful - responses to allowed inbound traffic are automatically allowed outbound and vice versa.

---

## Create a Security Group for a Web Server

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  # HTTP from anywhere
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS from anywhere
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH from trusted CIDR only
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }

  # All outbound traffic
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

---

## Security Group Rules as Separate Resources

```hcl
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_vpc_security_group_ingress_rule" "http" {
  security_group_id = aws_security_group.app.id
  from_port         = 8080
  to_port           = 8080
  ip_protocol       = "tcp"
  cidr_ipv4         = "0.0.0.0/0"
}

resource "aws_vpc_security_group_egress_rule" "all" {
  security_group_id = aws_security_group.app.id
  ip_protocol       = "-1"
  cidr_ipv4         = "0.0.0.0/0"
}
```

---

## Security Group Referencing Another SG

```hcl
resource "aws_security_group" "db" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]  # Only from web SG
  }
}
```

---

## Summary

Use `aws_security_group` with inline `ingress` and `egress` blocks, or the newer `aws_vpc_security_group_ingress_rule` and `aws_vpc_security_group_egress_rule` resources for individual rules that can be managed independently. Reference other security groups by ID in the `security_groups` argument to allow traffic between specific resource tiers without hardcoding IPs.
