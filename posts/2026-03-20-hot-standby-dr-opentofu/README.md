# How to Implement Hot Standby DR Strategy with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Disaster Recovery, Hot Standby, OpenTofu, AWS, Multi-Region, Active-Passive

Description: Learn how to implement the Hot Standby (Active-Passive) disaster recovery strategy using OpenTofu to maintain full-capacity mirror environment for near-instant failover.

## Overview

Hot Standby maintains a full-capacity, fully operational environment in the DR region that is kept synchronized with the primary region. Failover happens in seconds to minutes with minimal or no data loss. This is the most expensive DR strategy but provides the lowest RTO/RPO.

## Step 1: Identical Infrastructure in Both Regions

```hcl
# main.tf - Hot standby mirrors production exactly
module "app_primary" {
  source    = "./modules/app-environment"
  providers = { aws = aws.primary }

  environment    = "production-primary"
  desired_count  = 8
  instance_class = "db.r6g.2xlarge"
  min_instances  = 4
  max_instances  = 20
  region         = "us-east-1"
}

# Hot standby - same capacity as production
module "app_standby" {
  source    = "./modules/app-environment"
  providers = { aws = aws.dr }

  environment    = "production-standby"
  desired_count  = 8  # Same as primary - ready for immediate traffic
  instance_class = "db.r6g.2xlarge"  # Same DB class
  min_instances  = 4
  max_instances  = 20
  region         = "us-west-2"
}
```

## Step 2: Database with Synchronous Replication

```hcl
# Aurora Global Database for near-synchronous cross-region replication
resource "aws_rds_global_cluster" "app" {
  global_cluster_identifier = "app-global-db"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  database_name             = "appdb"
  storage_encrypted         = true
}

# Primary cluster
resource "aws_rds_cluster" "primary" {
  provider                  = aws.primary
  cluster_identifier        = "app-primary"
  engine                    = "aurora-postgresql"
  global_cluster_identifier = aws_rds_global_cluster.app.id

  # Aurora Global typically achieves < 1 second replication lag
  engine_mode = "provisioned"
}

# Hot standby cluster in DR region
resource "aws_rds_cluster" "standby" {
  provider                  = aws.dr
  cluster_identifier        = "app-standby"
  engine                    = "aurora-postgresql"
  global_cluster_identifier = aws_rds_global_cluster.app.id

  # Standby must be provisioned - cannot be serverless
  engine_mode = "provisioned"
}

resource "aws_rds_cluster_instance" "standby_instances" {
  provider           = aws.dr
  count              = 2
  cluster_identifier = aws_rds_cluster.standby.id
  instance_class     = "db.r6g.2xlarge"
  engine             = "aurora-postgresql"
}
```

## Step 3: Health Check with Automatic Failover

```hcl
# Route53 with health checks for sub-minute DNS failover
resource "aws_route53_health_check" "primary" {
  fqdn              = module.app_primary.alb_dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health/deep"  # Deep health check
  failure_threshold = 2  # Faster failover
  request_interval  = 10  # 10-second interval
}

resource "aws_route53_record" "primary_failover" {
  zone_id        = aws_route53_zone.app.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "primary"

  alias {
    name                   = module.app_primary.alb_dns_name
    zone_id                = module.app_primary.alb_zone_id
    evaluate_target_health = true
  }

  failover_routing_policy { type = "PRIMARY" }
  health_check_id = aws_route53_health_check.primary.id
}

resource "aws_route53_record" "standby_failover" {
  zone_id        = aws_route53_zone.app.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "standby"

  alias {
    name                   = module.app_standby.alb_dns_name
    zone_id                = module.app_standby.alb_zone_id
    evaluate_target_health = true
  }

  failover_routing_policy { type = "SECONDARY" }
  # No health check on secondary - it becomes active only when primary fails
}
```

## Step 4: Aurora Global Database Failover

```hcl
# Managed failover script (run when primary region is lost)
resource "null_resource" "global_db_failover" {
  count = var.execute_dr_failover ? 1 : 0

  provisioner "local-exec" {
    command = <<-EOT
      aws rds failover-global-cluster \
        --global-cluster-identifier ${aws_rds_global_cluster.app.id} \
        --target-db-cluster-identifier ${aws_rds_cluster.standby.arn} \
        --region us-west-2
    EOT
  }

  triggers = {
    execute = var.execute_dr_failover
  }
}
```

## Summary

Hot Standby DR implemented with OpenTofu provides RTO under 60 seconds using Route53 failover routing with 10-second health check intervals and a 20-second failure threshold. Aurora Global Database achieves typically under 1 second replication lag, providing near-zero RPO. The cost is approximately 2x production infrastructure, but this is justified for applications with strict SLAs requiring less than 5 minutes RTO and 1 minute RPO.
