# How to Configure VPC DNS Resolution with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPC, DNS, Route 53, Infrastructure as Code, Networking

Description: Learn how to configure VPC DNS resolution settings, Route 53 Resolver rules, and conditional forwarding for hybrid and multi-VPC architectures using OpenTofu.

## Introduction

AWS VPCs use the Amazon-provided DNS server at the VPC base CIDR plus 2 (e.g., 10.0.0.2). You can extend this with Route 53 Resolver endpoints and forwarding rules for split-horizon DNS and on-premises name resolution.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with VPC and Route 53 Resolver permissions

## Step 1: Enable DNS on the VPC

```hcl
# VPC must have both DNS support and hostnames enabled

# for instances to receive DNS names and resolve them
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true   # Enables the Amazon DNS server
  enable_dns_hostnames = true   # Assigns DNS hostnames to instances

  tags = { Name = "main-vpc" }
}
```

## Step 2: Create a Route 53 Private Hosted Zone

```hcl
# Private hosted zone for internal service discovery
resource "aws_route53_zone" "internal" {
  name    = "internal.example.com"
  comment = "Internal DNS for main VPC"

  vpc {
    vpc_id = aws_vpc.main.id
  }

  tags = { Name = "internal-dns-zone" }
}

# DNS record for an internal service
resource "aws_route53_record" "service" {
  zone_id = aws_route53_zone.internal.zone_id
  name    = "api.internal.example.com"
  type    = "A"
  ttl     = 300
  records = ["10.0.1.100"]
}
```

## Step 3: Create Route 53 Resolver Inbound Endpoint

```hcl
# Security group for the resolver endpoints
resource "aws_security_group" "resolver" {
  name   = "resolver-endpoint-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 53
    to_port     = 53
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }

  ingress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}

# Inbound endpoint allows on-premises DNS queries to resolve AWS names
resource "aws_route53_resolver_endpoint" "inbound" {
  name      = "inbound-resolver"
  direction = "INBOUND"

  security_group_ids = [aws_security_group.resolver.id]

  ip_address {
    subnet_id = var.subnet_id_az1
  }

  ip_address {
    subnet_id = var.subnet_id_az2
  }

  tags = { Name = "inbound-resolver" }
}
```

## Step 4: Create Route 53 Resolver Outbound Endpoint

```hcl
# Outbound endpoint allows AWS to forward queries to on-premises DNS
resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "outbound-resolver"
  direction = "OUTBOUND"

  security_group_ids = [aws_security_group.resolver.id]

  ip_address {
    subnet_id = var.subnet_id_az1
  }

  ip_address {
    subnet_id = var.subnet_id_az2
  }

  tags = { Name = "outbound-resolver" }
}
```

## Step 5: Create Resolver Forwarding Rules

```hcl
# Forward queries for corp.example.com to on-premises DNS servers
resource "aws_route53_resolver_rule" "forward_onprem" {
  domain_name          = "corp.example.com"
  name                 = "forward-to-onprem"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id

  target_ip {
    ip   = "192.168.1.10"
    port = 53
  }

  target_ip {
    ip   = "192.168.1.11"
    port = 53
  }
}

# Associate the forwarding rule with the VPC
resource "aws_route53_resolver_rule_association" "forward_onprem" {
  resolver_rule_id = aws_route53_resolver_rule.forward_onprem.id
  vpc_id           = aws_vpc.main.id
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

You have configured comprehensive DNS resolution for your VPC including private hosted zones, resolver endpoints, and forwarding rules. This setup supports hybrid DNS architectures where both AWS resources and on-premises servers can resolve names across the boundary, enabling seamless service discovery in multi-environment deployments.
