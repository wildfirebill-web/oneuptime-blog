# How to Set Up Active-Active Deployments with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: High Availability, Active-Active, OpenTofu, Route53, Global Accelerator, DynamoDB

Description: Learn how to configure active-active deployments using OpenTofu where multiple regions simultaneously serve traffic with automatic load balancing and failover.

## Overview

Active-active deployments run identical workloads in multiple regions simultaneously, routing users to the nearest region for low latency and providing instant failover. OpenTofu configures latency-based routing, data replication, and conflict resolution.

## Step 1: Latency-Based DNS Routing

```hcl
# main.tf - Route53 latency routing across regions
locals {
  regions = {
    "us-east-1" = {
      alb_dns  = module.app_us_east.alb_dns_name
      alb_zone = module.app_us_east.alb_zone_id
    }
    "eu-west-1" = {
      alb_dns  = module.app_eu_west.alb_dns_name
      alb_zone = module.app_eu_west.alb_zone_id
    }
    "ap-southeast-1" = {
      alb_dns  = module.app_ap_se.alb_dns_name
      alb_zone = module.app_ap_se.alb_zone_id
    }
  }
}

# Create health checks and DNS records for each region
resource "aws_route53_health_check" "region" {
  for_each = local.regions

  fqdn              = each.value.alb_dns
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "latency" {
  for_each = local.regions

  zone_id        = aws_route53_zone.app.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = each.key

  alias {
    name                   = each.value.alb_dns
    zone_id                = each.value.alb_zone
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = each.key
  }

  health_check_id = aws_route53_health_check.region[each.key].id
}
```

## Step 2: DynamoDB Global Tables (Write Anywhere)

```hcl
# DynamoDB Global Tables for active-active data replication
resource "aws_dynamodb_table" "active_active" {
  name             = "app-data-global"
  billing_mode     = "PAY_PER_REQUEST"
  hash_key         = "pk"
  range_key        = "sk"
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  attribute {
    name = "pk"
    type = "S"
  }

  attribute {
    name = "sk"
    type = "S"
  }

  # Replicas in all regions
  replica {
    region_name = "eu-west-1"
  }

  replica {
    region_name = "ap-southeast-1"
  }

  # Last-writer-wins conflict resolution (built-in)
  # For custom conflict resolution, use DynamoDB Streams + Lambda
}
```

## Step 3: Global Accelerator for Consistent Entry Points

```hcl
resource "aws_globalaccelerator_accelerator" "app" {
  name            = "app-global-accelerator"
  ip_address_type = "IPV4"
  enabled         = true
}

resource "aws_globalaccelerator_listener" "https" {
  accelerator_arn = aws_globalaccelerator_accelerator.app.id
  protocol        = "TCP"

  port_range {
    from_port = 443
    to_port   = 443
  }
}

# Endpoint groups per region with traffic distribution
resource "aws_globalaccelerator_endpoint_group" "us_east" {
  listener_arn          = aws_globalaccelerator_listener.https.id
  endpoint_group_region = "us-east-1"
  traffic_dial_percentage = 33.33  # Equal distribution

  endpoint_configuration {
    endpoint_id = module.app_us_east.alb_arn
    weight      = 100
    client_ip_preservation_enabled = true
  }

  health_check_path             = "/health"
  health_check_protocol         = "HTTPS"
  threshold_count               = 3
  health_check_interval_seconds = 30
}
```

## Step 4: Cache Invalidation Across Regions

```hcl
# ElastiCache Global Datastore for cross-region cache
resource "aws_elasticache_global_replication_group" "app" {
  global_replication_group_id_suffix = "app-cache"
  primary_replication_group_id       = aws_elasticache_replication_group.primary.id
  num_node_groups                    = 1
}

# Each region adds itself to the global datastore
resource "aws_elasticache_replication_group" "global_member_eu" {
  provider                    = aws.eu_west
  replication_group_id        = "app-cache-eu"
  description                 = "EU member of global cache"
  global_replication_group_id = aws_elasticache_global_replication_group.app.global_replication_group_id
  num_cache_clusters          = 2
}
```

## Summary

Active-active deployments configured with OpenTofu route users to their nearest region using Route53 latency routing with health checks, providing automatic failover when a region fails. DynamoDB Global Tables replicate writes across all regions using last-writer-wins conflict resolution, enabling any region to accept writes. Global Accelerator provides two static anycast IPs for DNS simplicity and uses AWS's backbone network to route from the user's entry point to the closest healthy endpoint.
