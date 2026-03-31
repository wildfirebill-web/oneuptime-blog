# How to Configure Redis Security Groups with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Terraform, Security Group, AWS, Network Security

Description: Learn how to create and manage AWS security groups for Redis using Terraform, restricting access to only authorized application tiers and IP ranges.

---

Security groups act as virtual firewalls for your AWS ElastiCache Redis cluster. Properly configuring them with Terraform ensures that only authorized resources can connect to Redis, reducing your attack surface significantly.

## Basic Security Group for Redis

The simplest pattern: allow access only from a known application security group:

```hcl
resource "aws_security_group" "redis" {
  name        = "redis-sg"
  description = "Security group for ElastiCache Redis"
  vpc_id      = var.vpc_id

  tags = {
    Name = "redis-sg"
  }
}

resource "aws_security_group_rule" "redis_ingress_from_app" {
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.app.id
  security_group_id        = aws_security_group.redis.id
  description              = "Allow Redis access from application tier"
}

resource "aws_security_group_rule" "redis_egress" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.redis.id
}
```

## Allowing Multiple Application Tiers

When you have multiple services (API, workers, batch jobs) that need Redis access:

```hcl
locals {
  app_security_groups = [
    aws_security_group.api.id,
    aws_security_group.worker.id,
    aws_security_group.batch.id,
  ]
}

resource "aws_security_group_rule" "redis_ingress" {
  count                    = length(local.app_security_groups)
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  source_security_group_id = local.app_security_groups[count.index]
  security_group_id        = aws_security_group.redis.id
  description              = "Redis access from app tier ${count.index}"
}
```

## Allowing Bastion Access for Administration

In non-production environments, allow bastion host access for debugging:

```hcl
resource "aws_security_group_rule" "redis_ingress_bastion" {
  count                    = var.environment == "production" ? 0 : 1
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion.id
  security_group_id        = aws_security_group.redis.id
  description              = "Redis access from bastion (non-prod only)"
}
```

## Using a Variable-Driven Module Pattern

Create a reusable module that accepts a list of allowed security group IDs:

```hcl
variable "allowed_security_group_ids" {
  type        = list(string)
  description = "List of security group IDs allowed to access Redis"
}

resource "aws_security_group_rule" "redis_ingress_dynamic" {
  for_each                 = toset(var.allowed_security_group_ids)
  type                     = "ingress"
  from_port                = 6379
  to_port                  = 6379
  protocol                 = "tcp"
  source_security_group_id = each.value
  security_group_id        = aws_security_group.redis.id
}
```

Call the module:

```hcl
module "redis_security" {
  source = "./modules/redis-security"
  vpc_id = module.vpc.vpc_id
  allowed_security_group_ids = [
    module.api.security_group_id,
    module.worker.security_group_id,
  ]
}
```

## Verifying Security Group Rules

```bash
terraform plan

# After apply, verify with AWS CLI
aws ec2 describe-security-groups \
  --group-ids sg-abc123 \
  --query 'SecurityGroups[0].IpPermissions' \
  --region us-east-1
```

Expected output:

```text
[
    {
        "IpProtocol": "tcp",
        "FromPort": 6379,
        "ToPort": 6379,
        "UserIdGroupPairs": [
            {
                "GroupId": "sg-app123",
                "Description": "Allow Redis access from application tier"
            }
        ]
    }
]
```

## Summary

Terraform security group rules for Redis should follow the principle of least privilege - grant access only to the specific security groups that need it. Use `aws_security_group_rule` as separate resources rather than inline rules for flexibility, and use `for_each` when managing access from multiple sources. Never allow Redis access from `0.0.0.0/0`.
