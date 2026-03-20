# How to Import AWS Security Groups into OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, AWS, Security Groups, Import, Networking

Description: Learn how to import existing AWS security groups and their rules into OpenTofu state, handling both inline rules and separate rule resources.

## Introduction

Security groups can be imported into OpenTofu as a single resource (with inline ingress/egress rules) or as separate `aws_security_group_rule` resources. The approach you choose depends on how you want to manage rules going forward.

## Approach 1: Import as Single Resource with Inline Rules

```bash
# Get security group ID and rules

SG_ID="sg-0123456789abcdef0"

aws ec2 describe-security-groups --group-ids $SG_ID \
  --query 'SecurityGroups[0]' --output json | jq '{
    group_name: .GroupName,
    description: .Description,
    vpc_id: .VpcId,
    ingress: [.IpPermissions[] | {
      from_port: .FromPort,
      to_port: .ToPort,
      protocol: .IpProtocol,
      cidr_blocks: [.IpRanges[].CidrIp],
      description: .IpRanges[0].Description
    }]
  }'
```

```hcl
resource "aws_security_group" "app" {
  name        = "app-server-sg"
  description = "Security group for application servers"
  vpc_id      = "vpc-0123456789abcdef0"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from internet"
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
    description = "App port from internal network"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound"
  }

  tags = { Name = "app-server-sg", Environment = "prod" }
}

import {
  to = aws_security_group.app
  id = "sg-0123456789abcdef0"
}
```

## Approach 2: Import Separate Security Group Rule Resources

This approach is useful when rules are managed by different teams or configurations:

```hcl
resource "aws_security_group" "app" {
  name        = "app-server-sg"
  description = "Security group for application servers"
  vpc_id      = "vpc-0123456789abcdef0"
  # No inline ingress/egress - managed as separate resources
  tags = { Name = "app-server-sg" }
}

resource "aws_security_group_rule" "https_ingress" {
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.app.id
  description       = "HTTPS from internet"
}

resource "aws_security_group_rule" "all_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.app.id
}
```

```hcl
# Security group rule IDs use composite format:
# SECURITY_GROUP_ID_TYPE_PROTOCOL_FROM_PORT_TO_PORT_CIDR
import {
  to = aws_security_group.app
  id = "sg-0123456789abcdef0"
}

import {
  to = aws_security_group_rule.https_ingress
  id = "sg-0123456789abcdef0_ingress_tcp_443_443_0.0.0.0/0"
}
```

## Handling Security Groups with Source SG References

```hcl
resource "aws_security_group_rule" "from_alb" {
  type                     = "ingress"
  from_port                = 8080
  to_port                  = 8080
  protocol                 = "tcp"
  source_security_group_id = "sg-alb0123456789abcdef"
  security_group_id        = aws_security_group.app.id
  description              = "Traffic from ALB"
}

# Import ID for source-SG rules uses the source SG ID instead of CIDR
import {
  to = aws_security_group_rule.from_alb
  id = "sg-0123456789abcdef0_ingress_tcp_8080_8080_sg-alb0123456789abcdef"
}
```

## Conclusion

Choose the inline rules approach when a single team manages all rules for a security group. Choose the separate `aws_security_group_rule` approach when rules come from multiple sources. Note that you cannot mix inline rules in `aws_security_group` with separate `aws_security_group_rule` resources - pick one approach per security group.
