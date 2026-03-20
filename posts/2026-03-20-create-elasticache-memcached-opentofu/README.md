# How to Create AWS ElastiCache Memcached with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ElastiCache, Memcached, Infrastructure as Code

Description: Learn how to create AWS ElastiCache Memcached clusters with OpenTofu for simple, high-performance in-memory caching with horizontal scalability.

ElastiCache for Memcached is ideal for simple caching use cases that need horizontal scaling across multiple nodes. Unlike Redis, Memcached is multi-threaded and scales by adding nodes without complex replication topology. Managing clusters in OpenTofu ensures consistent configuration.

## Creating a Memcached Cluster

```hcl
resource "aws_elasticache_subnet_group" "main" {
  name       = "memcached-subnet-group"
  subnet_ids = var.private_subnet_ids
}

resource "aws_elasticache_cluster" "memcached" {
  cluster_id           = "myapp-memcached"
  engine               = "memcached"
  node_type            = "cache.r7g.large"
  num_cache_nodes      = 3        # Total number of nodes (no primary/replica)
  parameter_group_name = aws_elasticache_parameter_group.memcached.name
  engine_version       = "1.6.22"
  port                 = 11211

  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.memcached.id]

  # Spread nodes across AZs
  az_mode = "cross-az"  # or "single-az"
  preferred_availability_zones = [
    "us-east-1a",
    "us-east-1b",
    "us-east-1c",
  ]

  maintenance_window = "sun:04:00-sun:05:00"

  tags = {
    Environment = "production"
    Team        = "backend"
  }
}
```

## Parameter Group

```hcl
resource "aws_elasticache_parameter_group" "memcached" {
  family = "memcached1.6"
  name   = "myapp-memcached"

  parameter {
    name  = "max_item_size"
    value = "10485760"  # 10 MB max item size
  }

  parameter {
    name  = "chunk_size_growth_factor"
    value = "1.25"
  }
}
```

## Security Group

```hcl
resource "aws_security_group" "memcached" {
  name        = "memcached-sg"
  description = "Memcached security group"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 11211
    to_port         = 11211
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }
}
```

## Auto Discovery Endpoint

Memcached clients use the Auto Discovery endpoint to automatically discover all cluster nodes:

```hcl
output "configuration_endpoint" {
  description = "Use this endpoint for auto-discovery of all nodes"
  value       = aws_elasticache_cluster.memcached.configuration_endpoint
}

output "cache_nodes" {
  description = "Individual node endpoints (for manual node management)"
  value = aws_elasticache_cluster.memcached.cache_nodes
}
```

## Scaling Memcached

```hcl
# Development cluster — 2 nodes
resource "aws_elasticache_cluster" "memcached_dev" {
  cluster_id      = "myapp-memcached-dev"
  engine          = "memcached"
  node_type       = "cache.t4g.small"
  num_cache_nodes = 2
  engine_version  = "1.6.22"
  port            = 11211

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.memcached.id]
}

# Production cluster — scale to 6 nodes
resource "aws_elasticache_cluster" "memcached_prod" {
  cluster_id      = "myapp-memcached-prod"
  engine          = "memcached"
  node_type       = "cache.r7g.xlarge"
  num_cache_nodes = 6
  engine_version  = "1.6.22"
  port            = 11211
  az_mode         = "cross-az"

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.memcached.id]
}
```

## CloudWatch Alarms

```hcl
resource "aws_cloudwatch_metric_alarm" "cache_miss_rate" {
  alarm_name          = "memcached-high-miss-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5
  metric_name         = "CacheMisses"
  namespace           = "AWS/ElastiCache"
  period              = 60
  statistic           = "Sum"
  threshold           = 1000
  alarm_description   = "High cache miss rate — consider increasing cache size"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    CacheClusterId = aws_elasticache_cluster.memcached.cluster_id
  }
}
```

## Conclusion

ElastiCache Memcached in OpenTofu provides simple, horizontally scalable caching. Use cross-AZ placement for resilience, configure the Auto Discovery endpoint in your application clients to automatically detect node additions and removals, and monitor cache miss rates to determine when to add nodes or increase node size.
