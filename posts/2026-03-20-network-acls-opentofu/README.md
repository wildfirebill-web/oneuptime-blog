# How to Manage Network ACLs with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Network ACLs, VPC Security, Infrastructure as Code

Description: Learn how to create and manage AWS Network Access Control Lists (NACLs) using OpenTofu to add a stateless layer of subnet-level network security.

## Introduction

Network ACLs (NACLs) are stateless subnet-level firewalls in AWS VPCs. Unlike Security Groups, which are stateful and instance-level, NACLs evaluate both inbound and outbound traffic independently. Managing NACLs with OpenTofu ensures consistent, version-controlled subnet access control across your environments.

## NACLs vs. Security Groups

| Feature | NACLs | Security Groups |
|---|---|---|
| Level | Subnet | Instance/ENI |
| Statefulness | Stateless | Stateful |
| Rules | Allow and Deny | Allow only |
| Rule evaluation | Numbered, lowest first | All rules evaluated |
| Return traffic | Must be explicitly allowed | Automatically allowed |

## Creating a Custom NACL

```hcl
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  tags = {
    Name        = "public-nacl"
    Environment = var.environment
  }
}
```

## Adding Inbound Rules

```hcl
# Allow HTTPS inbound from anywhere
resource "aws_network_acl_rule" "allow_https_in" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}

# Allow HTTP inbound from anywhere
resource "aws_network_acl_rule" "allow_http_in" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 110
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
}

# Allow ephemeral ports for return traffic (stateless requirement)
resource "aws_network_acl_rule" "allow_ephemeral_in" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 900
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}

# Deny all other inbound traffic
resource "aws_network_acl_rule" "deny_all_in" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 32766
  egress         = false
  protocol       = "-1"
  rule_action    = "deny"
  cidr_block     = "0.0.0.0/0"
  from_port      = 0
  to_port        = 0
}
```

## Adding Outbound Rules

```hcl
# Allow all outbound (typical for public subnets)
resource "aws_network_acl_rule" "allow_all_out" {
  network_acl_id = aws_network_acl.public.id
  rule_number    = 100
  egress         = true
  protocol       = "-1"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 0
  to_port        = 0
}
```

## Private Subnet NACL

For private subnets, restrict ingress to internal traffic only:

```hcl
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id

  tags = { Name = "private-nacl" }
}

# Allow inbound from VPC CIDR only
resource "aws_network_acl_rule" "allow_vpc_in" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 100
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = aws_vpc.main.cidr_block
  from_port      = 0
  to_port        = 65535
}

# Allow ephemeral ports from internet (for NAT gateway return traffic)
resource "aws_network_acl_rule" "allow_nat_return" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 200
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}
```

## Using Dynamic Blocks for Multiple Rules

```hcl
locals {
  inbound_rules = [
    { number = 100, port = 443, cidr = "0.0.0.0/0" },
    { number = 110, port = 80,  cidr = "0.0.0.0/0" },
    { number = 120, port = 22,  cidr = "10.0.0.0/8" },
  ]
}

resource "aws_network_acl" "app" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.app[*].id

  dynamic "ingress" {
    for_each = local.inbound_rules
    content {
      rule_no    = ingress.value.number
      action     = "allow"
      protocol   = "tcp"
      cidr_block = ingress.value.cidr
      from_port  = ingress.value.port
      to_port    = ingress.value.port
    }
  }

  tags = { Name = "app-nacl" }
}
```

## Best Practices

- Always allow ephemeral ports (1024-65535) for inbound on stateless NACLs.
- Use NACLs as a coarse-grained boundary; use Security Groups for fine-grained control.
- Number rules in increments of 10 or 100 to leave room for insertions.
- Keep the default VPC NACL permissive and use custom NACLs for specific subnets.
- Document the purpose of each rule number in resource tags or comments.

## Conclusion

Network ACLs in OpenTofu provide a stateless, subnet-level security layer complementing Security Groups. By managing NACLs as code, you ensure consistent network security boundaries across all environments and maintain a clear audit trail of all changes.
