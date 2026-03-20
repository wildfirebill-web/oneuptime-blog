# How to Configure Network ACLs on AWS with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Network ACLs, NACLs, AWS, Security, Networking, Infrastructure as Code

Description: Learn how to configure AWS Network ACLs (NACLs) using OpenTofu - understanding stateless rules, ephemeral ports, and how to layer NACLs with security groups for defense in depth.

## Introduction

Network ACLs are stateless subnet-level firewalls that process rules in order by rule number. Unlike security groups (stateful), NACLs require both inbound and outbound rules for each traffic flow, including ephemeral ports. OpenTofu manages NACLs with `aws_network_acl` and individual rules with `aws_network_acl_rule`.

## Default NACL vs Custom NACL

```hcl
# The default NACL allows all traffic - not recommended for production

# Create custom NACLs for each subnet tier

# Public subnet NACL
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  tags = {
    Name        = "${var.environment}-public-nacl"
    Environment = var.environment
  }
}

# Private subnet NACL
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id

  tags = { Name = "${var.environment}-private-nacl" }
}
```

## Public NACL Rules

```hcl
# Inbound: Allow HTTP from internet
resource "aws_network_acl_rule" "public_inbound_http" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = false  # Inbound
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
}

# Inbound: Allow HTTPS from internet
resource "aws_network_acl_rule" "public_inbound_https" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 110
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = false
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}

# Inbound: Allow ephemeral ports (return traffic for outbound connections)
resource "aws_network_acl_rule" "public_inbound_ephemeral" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 900
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = false
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024  # Ephemeral port range
  to_port        = 65535
}

# Outbound: Allow HTTP/HTTPS to internet
resource "aws_network_acl_rule" "public_outbound_http" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = true  # Outbound
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
}

resource "aws_network_acl_rule" "public_outbound_https" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 110
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = true
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}

# Outbound: Return traffic (ephemeral ports)
resource "aws_network_acl_rule" "public_outbound_ephemeral" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 900
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = true
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}
```

## Private NACL Rules

```hcl
# Private subnets: only allow traffic from VPC CIDR
resource "aws_network_acl_rule" "private_inbound_vpc" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = false
  cidr_block     = aws_vpc.main.cidr_block  # Only from within VPC
  from_port      = 0
  to_port        = 65535
}

# Allow HTTPS outbound to internet (via NAT Gateway)
resource "aws_network_acl_rule" "private_outbound_https" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = true
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}

# Allow ephemeral ports outbound (return traffic)
resource "aws_network_acl_rule" "private_outbound_ephemeral" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 900
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = true
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}

# Explicit deny for specific blocked CIDRs
resource "aws_network_acl_rule" "private_deny_blocked" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 1  # Lowest number = highest priority
  protocol       = "-1"
  rule_action    = "deny"
  egress         = false
  cidr_block     = var.blocked_cidr  # e.g., known malicious IP range
}
```

## Dynamic NACL Rules from Variable

```hcl
variable "allowed_ingress_ports" {
  default = [
    { port = 80,  cidr = "0.0.0.0/0" },
    { port = 443, cidr = "0.0.0.0/0" }
  ]
}

resource "aws_network_acl_rule" "public_inbound_dynamic" {
  count = length(var.allowed_ingress_ports)

  network_acl_id = aws_network_acl.public.id
  rule_number    = 100 + count.index * 10
  protocol       = "tcp"
  rule_action    = "allow"
  egress         = false
  cidr_block     = var.allowed_ingress_ports[count.index].cidr
  from_port      = var.allowed_ingress_ports[count.index].port
  to_port        = var.allowed_ingress_ports[count.index].port
}
```

## Conclusion

NACLs add subnet-level defense-in-depth alongside security groups. Key points: NACLs are stateless (require both inbound and outbound rules), always allow ephemeral ports (1024-65535) for return traffic, process rules in order (lowest rule number first), and apply to entire subnets. Use NACLs to block known malicious CIDRs, enforce subnet-tier isolation (public vs. private), and provide an additional layer of protection that security groups cannot provide due to their stateful nature.
