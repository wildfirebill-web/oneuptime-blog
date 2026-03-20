# How to Build a Multi-Region Active-Passive Setup with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Multi-Region, Active-Passive, Disaster Recovery, Route53, Infrastructure as Code

Description: Learn how to build a multi-region active-passive infrastructure on AWS with OpenTofu, including Route53 failover routing, cross-region RDS read replica, and automated failover.

## Introduction

An active-passive multi-region setup runs production workloads in a primary region while maintaining a warm standby in a secondary region. When the primary fails, traffic shifts to the secondary. This guide builds the infrastructure for active-passive failover using Route53 health checks, cross-region RDS replication, and S3 cross-region replication.

## Multi-Region Provider Configuration

```hcl
provider "aws" {
  region = "us-east-1"
  alias  = "primary"
}

provider "aws" {
  region = "us-west-2"
  alias  = "secondary"
}
```

## Route53 Failover Routing

```hcl
resource "aws_route53_health_check" "primary" {
  fqdn              = aws_lb.primary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = { Name = "primary-health-check" }
}

resource "aws_route53_record" "primary" {
  zone_id = var.hosted_zone_id
  name    = var.domain_name
  type    = "A"

  set_identifier = "primary"
  health_check_id = aws_route53_health_check.primary.id

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "secondary" {
  zone_id = var.hosted_zone_id
  name    = var.domain_name
  type    = "A"

  set_identifier = "secondary"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

## Cross-Region RDS Read Replica

```hcl
# Primary database in us-east-1
resource "aws_db_instance" "primary" {
  provider    = aws.primary
  identifier  = "myapp-primary-db"
  engine      = "postgres"
  multi_az    = true  # HA within primary region
  # ... other config
}

# Read replica in us-west-2 for failover
resource "aws_db_instance" "secondary" {
  provider   = aws.secondary
  identifier = "myapp-secondary-db"

  # Replicate from primary
  replicate_source_db = aws_db_instance.primary.arn

  # Warm standby settings
  multi_az           = false  # promote to true during failover
  publicly_accessible = false

  lifecycle {
    prevent_destroy = true
    ignore_changes  = [replicate_source_db]  # changes during failover
  }
}
```

## S3 Cross-Region Replication

```hcl
resource "aws_s3_bucket" "primary_assets" {
  provider = aws.primary
  bucket   = "myapp-assets-primary"
}

resource "aws_s3_bucket" "secondary_assets" {
  provider = aws.secondary
  bucket   = "myapp-assets-secondary"
}

resource "aws_s3_bucket_replication_configuration" "assets" {
  provider = aws.primary
  role     = aws_iam_role.replication.arn
  bucket   = aws_s3_bucket.primary_assets.id

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.secondary_assets.arn
      storage_class = "STANDARD_IA"  # cheaper for standby
    }
  }
}
```

## ECS in Secondary Region (Warm Standby)

```hcl
# Secondary ECS service runs with minimum capacity
resource "aws_ecs_service" "secondary" {
  provider        = aws.secondary
  name            = "myapp-secondary"
  cluster         = aws_ecs_cluster.secondary.id
  task_definition = aws_ecs_task_definition.secondary.arn
  desired_count   = 1  # warm standby, scale up during failover
  launch_type     = "FARGATE"
  # ... other config
}
```

## Failover Runbook

```bash
# Automated failover steps (triggered by Route53 health check failure):

# 1. Route53 automatically shifts traffic to secondary

# 2. Scale up secondary ECS service
aws ecs update-service \
  --cluster myapp-secondary-cluster \
  --service myapp-secondary \
  --desired-count 5 \
  --region us-west-2

# 3. Promote secondary RDS to standalone (break replication)
aws rds promote-read-replica \
  --db-instance-identifier myapp-secondary-db \
  --region us-west-2

# 4. Update secondary to connect to promoted RDS
```

## Summary

An active-passive multi-region setup uses Route53 failover routing with health checks for automatic DNS failover, cross-region RDS read replicas for database standby, S3 cross-region replication for asset availability, and warm ECS standby in the secondary region. The Route53 health check automatically shifts traffic when the primary fails — the secondary ECS service needs to scale up and the RDS replica needs to be promoted. Automate these steps with Lambda triggered by Route53 health check state changes.
