# How to Configure IPv6 for AWS ElastiCache

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, IPv6, ElastiCache, Redis, Memcached, Cache, Dual-Stack

Description: Enable dual-stack mode on AWS ElastiCache clusters for Redis and Memcached to support IPv6 cache connections from applications in dual-stack or IPv6-only VPC subnets.

## Introduction

AWS ElastiCache supports IPv6 connections through dual-stack mode for both Redis and Memcached engines. When configured with dual-stack, the cache cluster creates endpoints that return both IPv4 and IPv6 addresses, enabling applications in IPv6-only subnets to connect to the cache. This is important for modern AWS architectures using IPv6-native EKS clusters or IPv6-only VPC deployments.

## Create ElastiCache Subnet Group with IPv6

```bash
# Create subnet group with IPv6-enabled subnets

aws elasticache create-cache-subnet-group \
    --cache-subnet-group-name ipv6-cache-subnet-group \
    --cache-subnet-group-description "IPv6 cache subnet group" \
    --subnet-ids subnet-ipv6-a subnet-ipv6-b subnet-ipv6-c
```

## Create Redis Cluster with Dual-Stack

```bash
# Create Redis cluster with dual-stack
aws elasticache create-replication-group \
    --replication-group-id my-redis-cluster \
    --description "Redis with IPv6 support" \
    --cache-node-type cache.t3.micro \
    --engine redis \
    --engine-version "7.0" \
    --num-cache-clusters 2 \
    --cache-subnet-group-name ipv6-cache-subnet-group \
    --security-group-ids sg-12345678 \
    --ip-discovery DUAL_STACK \
    --network-type DUAL_STACK

# Get the cluster endpoint
aws elasticache describe-replication-groups \
    --replication-group-id my-redis-cluster \
    --query "ReplicationGroups[0].{ConfigurationEndpoint:ConfigurationEndpoint, NodeGroups:NodeGroups}"
```

## Terraform ElastiCache with IPv6

```hcl
# elasticache_ipv6.tf

resource "aws_elasticache_subnet_group" "main" {
  name       = "main-cache-subnet-group"
  subnet_ids = [
    aws_subnet.private_a.id,  # IPv6-enabled subnets
    aws_subnet.private_b.id,
    aws_subnet.private_c.id,
  ]

  tags = { Name = "main-cache-subnet-group" }
}

# Redis Replication Group with dual-stack
resource "aws_elasticache_replication_group" "redis" {
  replication_group_id = "main-redis"
  description          = "Redis cluster with IPv6"

  engine         = "redis"
  engine_version = "7.0"
  node_type      = "cache.t3.micro"

  num_cache_clusters = 2
  port               = 6379

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  # Enable dual-stack (IPv4 + IPv6)
  network_type = "dual_stack"
  ip_discovery = "ipv6"

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true

  automatic_failover_enabled = true

  tags = { Name = "redis-dual-stack" }
}

# Memcached cluster with dual-stack
resource "aws_elasticache_cluster" "memcached" {
  cluster_id           = "main-memcached"
  engine               = "memcached"
  engine_version       = "1.6.17"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 2
  parameter_group_name = "default.memcached1.6"
  port                 = 11211

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.memcached.id]

  # Enable dual-stack
  network_type = "dual_stack"
  ip_discovery = "ipv6"

  tags = { Name = "memcached-dual-stack" }
}

# Security group for Redis
resource "aws_security_group" "redis" {
  vpc_id = aws_vpc.main.id
  name   = "redis-sg"

  # Allow Redis from app servers (IPv4)
  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # Allow Redis over IPv6 from VPC
  ingress {
    from_port        = 6379
    to_port          = 6379
    protocol         = "tcp"
    ipv6_cidr_blocks = [aws_vpc.main.ipv6_cidr_block]
    description      = "Redis over IPv6 from VPC"
  }
}
```

## Connect to ElastiCache over IPv6

```python
import redis
import socket

# Verify the endpoint resolves to IPv6
endpoint = "clustercfg.my-redis-cluster.abc.use1.cache.amazonaws.com"
addrs = socket.getaddrinfo(endpoint, 6379, socket.AF_INET6)
print(f"IPv6 endpoints: {[a[4][0] for a in addrs]}")

# Connect to Redis (will use IPv6 if preferred)
r = redis.Redis(
    host=endpoint,
    port=6379,
    ssl=True,
    ssl_cert_reqs="required",
    ssl_ca_certs="/etc/ssl/certs/ca-certificates.crt",
    socket_connect_timeout=5
)

# Test connection
r.set("test-key", "hello-ipv6")
value = r.get("test-key")
print(f"Value: {value.decode()}")
```

## Conclusion

ElastiCache dual-stack mode (`network_type = "dual_stack"`) enables both IPv4 and IPv6 connections to Redis and Memcached clusters. Configure subnet groups with IPv6-enabled subnets and set `ip_discovery = "ipv6"` to return IPv6 addresses from cluster discovery. Security groups must include IPv6 rules for the cache port (6379 for Redis, 11211 for Memcached). Applications connect using the standard endpoint DNS name, which resolves to IPv6 when in a dual-stack or IPv6-only environment.
