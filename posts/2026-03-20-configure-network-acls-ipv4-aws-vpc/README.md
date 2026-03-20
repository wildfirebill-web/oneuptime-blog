# How to Configure Network ACLs for IPv4 in AWS VPC

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, VPC, Network ACLs, IPv4, Security, OpenTofu

Description: Learn how to create and configure stateless Network ACLs in AWS VPC to control IPv4 traffic at the subnet boundary.

---

Network ACLs (NACLs) are stateless firewall rules applied at the subnet level in AWS VPC. Unlike security groups, NACLs process both inbound and outbound rules independently and evaluate rules in order by rule number.

---

## Create a Network ACL with OpenTofu

```hcl
resource "aws_network_acl" "app" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "app-nacl"
  }
}
```

---

## Add Inbound Rules

```hcl
# Allow HTTP from anywhere
resource "aws_network_acl_rule" "inbound_http" {
  network_acl_id = aws_network_acl.app.id
  rule_number    = 100
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
}

# Allow HTTPS from anywhere
resource "aws_network_acl_rule" "inbound_https" {
  network_acl_id = aws_network_acl.app.id
  rule_number    = 110
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 443
  to_port        = 443
}

# Allow ephemeral ports (response traffic — NACLs are stateless)
resource "aws_network_acl_rule" "inbound_ephemeral" {
  network_acl_id = aws_network_acl.app.id
  rule_number    = 900
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}

# Deny everything else
resource "aws_network_acl_rule" "inbound_deny_all" {
  network_acl_id = aws_network_acl.app.id
  rule_number    = 32766
  egress         = false
  protocol       = "-1"
  rule_action    = "deny"
  cidr_block     = "0.0.0.0/0"
}
```

---

## Add Outbound Rules

```hcl
resource "aws_network_acl_rule" "outbound_all" {
  network_acl_id = aws_network_acl.app.id
  rule_number    = 100
  egress         = true
  protocol       = "-1"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
}
```

---

## Associate with Subnets

```hcl
resource "aws_network_acl_association" "app" {
  count          = length(aws_subnet.app)
  network_acl_id = aws_network_acl.app.id
  subnet_id      = aws_subnet.app[count.index].id
}
```

---

## Key Differences: NACLs vs Security Groups

| Feature          | NACL                        | Security Group          |
|------------------|-----------------------------|-------------------------|
| Level            | Subnet                      | Instance/ENI            |
| Statefulness     | Stateless                   | Stateful                |
| Rule evaluation  | In order (lowest first)     | All rules evaluated     |
| Default action   | Deny if no rule matches     | Deny if no rule matches |

---

## Summary

NACLs are stateless — you must allow both request and response traffic explicitly. Always include an ephemeral port range (1024-65535) in inbound rules for return TCP traffic. Associate the NACL with subnets using `aws_network_acl_association`. Use NACLs as an additional layer of defense alongside security groups, not as a replacement.
