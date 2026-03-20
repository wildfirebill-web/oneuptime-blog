# How to Build a Multi-Region Active-Active Architecture with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: AWS, Multi-Region, Active-Active, OpenTofu, Route53, DynamoDB Global Tables, High Availability

Description: Learn how to build a multi-region active-active architecture on AWS using OpenTofu with Route53 latency routing, DynamoDB Global Tables, and Global Accelerator.

## Overview

An active-active multi-region architecture runs workloads in multiple AWS regions simultaneously, routing users to the nearest region for low latency and providing automatic failover if a region becomes unavailable. OpenTofu uses provider aliases to manage resources across regions.

## Step 1: Multi-Region Provider Configuration

```hcl
# main.tf - Multi-region provider setup
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "secondary"
  region = "eu-west-1"
}

# DynamoDB Global Tables replicate across regions
resource "aws_dynamodb_table" "global" {
  provider         = aws.primary
  name             = "global-app-data"
  billing_mode     = "PAY_PER_REQUEST"
  hash_key         = "id"
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  attribute {
    name = "id"
    type = "S"
  }

  replica {
    region_name = "eu-west-1"
  }

  replica {
    region_name = "ap-southeast-1"
  }
}
```

## Step 2: Application Deployment in Each Region

```hcl
# Deploy ECS service in primary region
module "app_primary" {
  source = "./modules/app-service"
  providers = { aws = aws.primary }

  cluster_name = "app-us-east-1"
  region       = "us-east-1"
  vpc_id       = module.vpc_primary.vpc_id
  subnet_ids   = module.vpc_primary.private_subnets
}

# Deploy same application in secondary region
module "app_secondary" {
  source = "./modules/app-service"
  providers = { aws = aws.secondary }

  cluster_name = "app-eu-west-1"
  region       = "eu-west-1"
  vpc_id       = module.vpc_secondary.vpc_id
  subnet_ids   = module.vpc_secondary.private_subnets
}
```

## Step 3: Route53 Latency-Based Routing

```hcl
# Route53 latency routing to nearest region
resource "aws_route53_record" "app_primary" {
  provider = aws.primary
  zone_id  = aws_route53_zone.app.zone_id
  name     = "app.example.com"
  type     = "A"

  alias {
    name                   = module.app_primary.alb_dns_name
    zone_id                = module.app_primary.alb_zone_id
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = "us-east-1"
  }

  health_check_id = aws_route53_health_check.primary.id
  set_identifier  = "primary"
}

resource "aws_route53_record" "app_secondary" {
  provider = aws.secondary
  zone_id  = aws_route53_zone.app.zone_id
  name     = "app.example.com"
  type     = "A"

  alias {
    name                   = module.app_secondary.alb_dns_name
    zone_id                = module.app_secondary.alb_zone_id
    evaluate_target_health = true
  }

  latency_routing_policy {
    region = "eu-west-1"
  }

  health_check_id = aws_route53_health_check.secondary.id
  set_identifier  = "secondary"
}

# Health checks for automatic failover
resource "aws_route53_health_check" "primary" {
  fqdn              = module.app_primary.alb_dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}
```

## Step 4: Global Accelerator for Consistent Entry Points

```hcl
# AWS Global Accelerator for static IP and anycast routing
resource "aws_globalaccelerator_accelerator" "app" {
  name            = "app-accelerator"
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

resource "aws_globalaccelerator_endpoint_group" "primary" {
  listener_arn          = aws_globalaccelerator_listener.https.id
  endpoint_group_region = "us-east-1"
  traffic_dial_percentage = 50  # Split traffic 50/50

  endpoint_configuration {
    endpoint_id                    = module.app_primary.alb_arn
    weight                         = 100
    client_ip_preservation_enabled = true
  }
}
```

## Summary

A multi-region active-active architecture on AWS built with OpenTofu provides sub-100ms global latency by routing users to their nearest region. DynamoDB Global Tables replicate data across regions with conflict resolution, Route53 latency routing with health checks provides automatic failover when a region degrades, and Global Accelerator uses AWS's backbone network to optimize routing from any point on the globe.
