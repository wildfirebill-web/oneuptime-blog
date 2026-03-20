# How to Configure Cache Failover with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ElastiCache, Failover, High Availability, Infrastructure as Code

Description: Learn how to configure ElastiCache automatic failover with OpenTofu to ensure cache availability when primary nodes fail.

ElastiCache failover automatically promotes a replica to primary when the primary node fails. Configuring this in OpenTofu ensures your HA settings are version-controlled and consistently applied, preventing manual misconfiguration of production caches.

## Enabling Automatic Failover

```hcl
resource "aws_elasticache_replication_group" "ha" {
  replication_group_id = "ha-redis"
  description          = "Redis with automatic failover"

  engine_version = "7.1"
  node_type      = "cache.r7g.large"
  port           = 6379

  # Minimum 2 nodes required for automatic failover
  num_cache_clusters = 2  # 1 primary + 1 replica

  automatic_failover_enabled = true  # Enable automatic failover
  multi_az_enabled           = true  # Place nodes in different AZs

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}
```

## Multi-AZ with Preferred Zones

```hcl
resource "aws_elasticache_replication_group" "multi_az" {
  replication_group_id = "multi-az-redis"
  description          = "Redis spread across 3 AZs"

  engine_version     = "7.1"
  node_type          = "cache.r7g.large"
  num_cache_clusters = 3

  automatic_failover_enabled = true
  multi_az_enabled           = true

  # Explicitly place nodes in different AZs
  availability_zones = [
    "us-east-1a",  # Primary
    "us-east-1b",  # Replica 1
    "us-east-1c",  # Replica 2
  ]

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
}
```

## Failover Monitoring Alarms

```hcl
# Alert when a failover occurs
resource "aws_cloudwatch_metric_alarm" "failover_events" {
  alarm_name          = "redis-failover-occurred"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ReplicationGroupFailoverCount"
  namespace           = "AWS/ElastiCache"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Redis failover occurred — verify application recovery"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.ha.id
  }
}

# Alert on unhealthy nodes
resource "aws_cloudwatch_metric_alarm" "unhealthy_nodes" {
  alarm_name          = "redis-unhealthy-nodes"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HealthyHostCount"
  namespace           = "AWS/ElastiCache"
  period              = 60
  statistic           = "Minimum"
  threshold           = 2
  alarm_description   = "Redis has fewer healthy nodes than expected"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ReplicationGroupId = aws_elasticache_replication_group.ha.id
  }
}
```

## Connection Handling in Applications

When failover occurs, the primary endpoint DNS updates to point to the new primary. Applications should:

```hcl
# Output the primary endpoint — DNS updates automatically on failover
output "redis_primary_endpoint" {
  description = "Connect to this endpoint. DNS updates automatically during failover."
  value       = aws_elasticache_replication_group.ha.primary_endpoint_address
}

output "redis_reader_endpoint" {
  description = "Read endpoint load-balances across all replicas."
  value       = aws_elasticache_replication_group.ha.reader_endpoint_address
}
```

## Backup for Recovery

```hcl
resource "aws_elasticache_replication_group" "with_backup" {
  replication_group_id = "backed-up-redis"
  description          = "Redis with snapshots for recovery"

  engine_version     = "7.1"
  node_type          = "cache.r7g.large"
  num_cache_clusters = 2

  automatic_failover_enabled = true
  multi_az_enabled           = true

  # Snapshot settings
  snapshot_retention_limit = 7         # Keep 7 daily snapshots
  snapshot_window          = "02:00-03:00"  # UTC

  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
}
```

## Conclusion

ElastiCache automatic failover in OpenTofu ensures your cache is highly available with minimal configuration. Enable multi_az_enabled alongside automatic_failover_enabled to place the replica in a different AZ, set up CloudWatch alarms to notify your team when failover occurs, and ensure applications connect via the primary endpoint DNS (which updates automatically) rather than hardcoded IP addresses.
