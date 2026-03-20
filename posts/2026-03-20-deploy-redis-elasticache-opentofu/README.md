# How to Deploy Redis on AWS ElastiCache with OpenTofu - Elasticache

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ElastiCache, Redis, Cache, Infrastructure as Code

Description: Learn how to provision an AWS ElastiCache Redis cluster with subnet groups, parameter groups, and encryption using OpenTofu.

---

AWS ElastiCache for Redis provides a fully managed in-memory cache. OpenTofu manages the subnet group, parameter group, security group, and replication group for a production-ready Redis deployment.

---

## Create the Subnet Group

```hcl
resource "aws_elasticache_subnet_group" "redis" {
  name       = "redis-subnet-group"
  subnet_ids = aws_subnet.private[*].id
}
```

---

## Create a Parameter Group

```hcl
resource "aws_elasticache_parameter_group" "redis7" {
  name   = "redis7-custom"
  family = "redis7"

  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }

  parameter {
    name  = "timeout"
    value = "300"
  }
}
```

---

## Create the Redis Replication Group

```hcl
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "redis-cluster"
  description          = "Redis cache cluster"

  node_type            = "cache.t3.micro"
  num_cache_clusters   = 2  # 1 primary + 1 replica
  port                 = 6379

  parameter_group_name = aws_elasticache_parameter_group.redis7.name
  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [aws_security_group.redis.id]

  engine_version       = "7.0"
  at_rest_encryption_enabled  = true
  transit_encryption_enabled  = true
  auth_token                  = var.redis_auth_token

  automatic_failover_enabled = true

  tags = {
    Name = "redis-production"
  }
}
```

---

## Security Group for Redis

```hcl
resource "aws_security_group" "redis" {
  name   = "redis-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}
```

---

## Output the Endpoints

```hcl
output "redis_primary_endpoint" {
  value = aws_elasticache_replication_group.redis.primary_endpoint_address
}

output "redis_reader_endpoint" {
  value = aws_elasticache_replication_group.redis.reader_endpoint_address
}
```

---

## Summary

Create `aws_elasticache_replication_group` with `num_cache_clusters = 2` for a primary + replica setup. Enable `at_rest_encryption_enabled`, `transit_encryption_enabled`, and set an `auth_token` for security. Use the `primary_endpoint_address` for writes and `reader_endpoint_address` for reads. Configure `maxmemory-policy` in a parameter group to control eviction behavior.
