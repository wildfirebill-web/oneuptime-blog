# How to Implement Warm Standby DR Strategy with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Disaster Recovery, Warm Standby, OpenTofu, AWS, DR Strategy, RTO, RPO

Description: Learn how to implement the Warm Standby disaster recovery strategy using OpenTofu to maintain a reduced-capacity replica environment ready for rapid scale-up during failover.

## Overview

Warm Standby maintains a fully functional but reduced-capacity environment in the DR region. Unlike Pilot Light, the application tier runs continuously (at 25-50% capacity), enabling faster failover with lower RTO. OpenTofu manages both regions with capacity variables.

## Step 1: Warm Standby Configuration

```hcl
# main.tf - Warm standby runs at reduced capacity
locals {
  # Production capacity
  prod_min_size   = 4
  prod_max_size   = 20
  prod_desired    = 8
  prod_db_class   = "db.r6g.2xlarge"

  # Warm standby runs at 25% of production capacity
  dr_min_size     = 1
  dr_max_size     = 20  # Can scale to full production
  dr_desired      = 2   # Running but reduced
  dr_db_class     = "db.r6g.large"  # Smaller instance

  # During DR failover, match production
  failover_desired = var.dr_failover ? local.prod_desired : local.dr_desired
  failover_db_class = var.dr_failover ? local.prod_db_class : local.dr_db_class
}

variable "dr_failover" {
  type        = bool
  default     = false
  description = "Set to true to scale DR to full production capacity"
}
```

## Step 2: DR Application Tier (Always Running)

```hcl
# Application running in DR at reduced capacity
resource "aws_autoscaling_group" "warm_standby" {
  provider    = aws.dr
  name        = "app-warm-standby"
  vpc_zone_identifier = module.vpc_dr.private_subnets

  min_size         = local.dr_min_size
  max_size         = local.dr_max_size
  desired_capacity = local.failover_desired

  launch_template {
    id      = aws_launch_template.dr.id
    version = "$Latest"
  }

  health_check_type         = "ELB"
  health_check_grace_period = 300

  tag {
    key                 = "Environment"
    value               = "DR-Warm-Standby"
    propagate_at_launch = true
  }
}
```

## Step 3: DB Instance Sizing (Scale Up During Failover)

```hcl
# DR read replica - scales during failover
resource "aws_db_instance" "warm_standby" {
  provider            = aws.dr
  identifier          = "app-db-warm-standby"
  replicate_source_db = aws_db_instance.primary.arn

  # Smaller instance class in standby, promoted to match primary during DR
  instance_class = local.failover_db_class

  # Multi-AZ in DR for resilience when it becomes primary
  multi_az = var.dr_failover ? true : false
}
```

## Step 4: Route53 DNS Switching

```hcl
# Weighted routing: 100% to primary in normal ops, 100% to DR during failover
resource "aws_route53_record" "weighted_primary" {
  zone_id        = aws_route53_zone.app.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "primary"

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }

  weighted_routing_policy {
    weight = var.dr_failover ? 0 : 100
  }
}

resource "aws_route53_record" "weighted_dr" {
  zone_id        = aws_route53_zone.app.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "dr"

  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }

  weighted_routing_policy {
    weight = var.dr_failover ? 100 : 0
  }
}
```

## Summary

Warm Standby DR implemented with OpenTofu provides an RTO of 5-15 minutes by keeping the DR application tier continuously running at reduced capacity. During failover, `tofu apply -var="dr_failover=true"` scales the DR ASG to full production capacity, promotes the database replica, and shifts DNS weights from primary to DR. The continuous operation means no cold-start delays, only scale-out time, making Warm Standby suitable for applications requiring RTO under 30 minutes.
