# How to Set Up ElastiCache Redis VPC Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ElastiCache, AWS, VPC, Networking

Description: Learn how to configure ElastiCache Redis in a VPC with subnet groups, security groups, and routing so only authorized resources can access your cache cluster.

---

ElastiCache Redis must reside in a VPC - it cannot be deployed in the EC2-Classic network. Controlling access means getting the subnet group, security groups, and routing right.

## Step 1: Create a Cache Subnet Group

The subnet group tells ElastiCache which subnets (and AZs) to place nodes in. Use private subnets only:

```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name redis-private-subnets \
  --cache-subnet-group-description "Private subnets for Redis" \
  --subnet-ids subnet-0aaa111 subnet-0bbb222 subnet-0ccc333
```

## Step 2: Create a Redis Security Group

```bash
aws ec2 create-security-group \
  --group-name redis-sg \
  --description "ElastiCache Redis access" \
  --vpc-id vpc-0abc12345

# Store the returned SG ID
REDIS_SG=sg-0redis123
```

## Step 3: Allow Application Access

Only allow traffic from your application security groups - never open to 0.0.0.0/0:

```bash
# Allow app servers
aws ec2 authorize-security-group-ingress \
  --group-id $REDIS_SG \
  --protocol tcp \
  --port 6379 \
  --source-group sg-0app456

# Allow bastion (for admin access)
aws ec2 authorize-security-group-ingress \
  --group-id $REDIS_SG \
  --protocol tcp \
  --port 6379 \
  --source-group sg-0bastion789
```

## Step 4: Deploy the Cluster

```bash
aws elasticache create-replication-group \
  --replication-group-id vpc-redis \
  --replication-group-description "VPC-isolated Redis" \
  --engine redis \
  --cache-node-type cache.r7g.large \
  --num-cache-clusters 2 \
  --cache-subnet-group-name redis-private-subnets \
  --security-group-ids $REDIS_SG \
  --transit-encryption-enabled \
  --multi-az-enabled \
  --automatic-failover-enabled
```

## Terraform Example

```hcl
resource "aws_elasticache_subnet_group" "redis" {
  name       = "redis-private-subnets"
  subnet_ids = var.private_subnet_ids
}

resource "aws_security_group" "redis" {
  name   = "redis-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_elasticache_replication_group" "main" {
  replication_group_id       = "vpc-redis"
  description                = "VPC-isolated Redis"
  node_type                  = "cache.r7g.large"
  num_cache_clusters         = 2
  subnet_group_name          = aws_elasticache_subnet_group.redis.name
  security_group_ids         = [aws_security_group.redis.id]
  transit_encryption_enabled = true
}
```

## Verifying No Public Access

```bash
aws elasticache describe-replication-groups \
  --replication-group-id vpc-redis \
  --query "ReplicationGroups[0].{AtRest:AtRestEncryptionEnabled,InTransit:TransitEncryptionEnabled,SNSArn:SnsTopic}"
```

Confirm that there is no public endpoint - the cluster DNS should resolve to a private IP within your VPC CIDR.

## Summary

ElastiCache Redis VPC access relies on a cache subnet group targeting private subnets, a security group that whitelists only authorized source SGs, and routing that keeps traffic internal. Never expose port 6379 to public CIDRs. Combine VPC isolation with TLS and auth tokens for defense-in-depth security.
