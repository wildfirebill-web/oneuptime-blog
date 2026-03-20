# How to Set Up Route 53 Failover Routing with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Route 53, Failover, DNS, High Availability, Infrastructure as Code

Description: Learn how to configure Route 53 failover routing policies with OpenTofu to automatically redirect traffic to a backup endpoint when the primary becomes unhealthy.

## Introduction

Route 53 failover routing automatically directs traffic to a backup (secondary) endpoint when the primary endpoint's health check fails. This DNS-based failover achieves RPOs of 60-90 seconds for DNS TTL propagation and is commonly used for active-passive disaster recovery between regions or between primary services and maintenance pages.

## Prerequisites

- OpenTofu v1.6+
- Route 53 health checks for the primary endpoint
- A hosted zone in Route 53
- AWS credentials with Route 53 permissions

## Step 1: Create Primary and Secondary Records

```hcl
# Health check for the primary endpoint

resource "aws_route53_health_check" "primary" {
  fqdn              = var.primary_endpoint_fqdn
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name = "${var.project_name}-primary-health-check"
  }
}

# Primary record (active endpoint)
resource "aws_route53_record" "primary" {
  zone_id        = var.hosted_zone_id
  name           = var.domain_name
  type           = "A"
  set_identifier = "primary"

  failover_routing_policy {
    type = "PRIMARY"
  }

  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = var.primary_alb_dns_name
    zone_id                = var.primary_alb_zone_id
    evaluate_target_health = true
  }
}

# Secondary record (failover target)
# No health check required - Route 53 routes to this if primary is unhealthy
resource "aws_route53_record" "secondary" {
  zone_id        = var.hosted_zone_id
  name           = var.domain_name
  type           = "A"
  set_identifier = "secondary"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = var.secondary_alb_dns_name
    zone_id                = var.secondary_alb_zone_id
    evaluate_target_health = true
  }
}
```

## Step 2: Multi-Level Failover with S3 Static Page

```hcl
# Failover to an S3-hosted maintenance page when all dynamic endpoints fail
resource "aws_s3_bucket" "maintenance" {
  bucket = "${var.project_name}-maintenance-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_website_configuration" "maintenance" {
  bucket = aws_s3_bucket.maintenance.id

  index_document { suffix = "index.html" }
  error_document { key = "error.html" }
}

resource "aws_route53_record" "maintenance_fallback" {
  zone_id        = var.hosted_zone_id
  name           = var.domain_name
  type           = "A"
  set_identifier = "maintenance"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name    = aws_s3_bucket_website_configuration.maintenance.website_domain
    zone_id = var.s3_hosted_zone_id
    evaluate_target_health = false
  }
}
```

## Step 3: Cross-Region Failover

```hcl
# Primary region health check
resource "aws_route53_health_check" "us_east_1" {
  fqdn              = "us-east-1.api.${var.domain_name}"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10  # 10-second interval for faster detection

  regions = ["us-east-1", "eu-west-1", "ap-southeast-1"]
}

resource "aws_route53_record" "us_east_1_primary" {
  zone_id        = var.hosted_zone_id
  name           = "api.${var.domain_name}"
  type           = "A"
  set_identifier = "us-east-1"

  failover_routing_policy {
    type = "PRIMARY"
  }

  health_check_id = aws_route53_health_check.us_east_1.id

  alias {
    name                   = var.us_east_1_alb_dns
    zone_id                = var.us_east_1_alb_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "eu_west_1_secondary" {
  zone_id        = var.hosted_zone_id
  name           = "api.${var.domain_name}"
  type           = "A"
  set_identifier = "eu-west-1"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = var.eu_west_1_alb_dns
    zone_id                = var.eu_west_1_alb_zone_id
    evaluate_target_health = true
  }
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply

# Test failover by checking current DNS resolution
dig api.example.com

# Simulate failover by manually failing the health check
aws route53 update-health-check \
  --health-check-id <id> \
  --disabled  # Temporarily disable to test failover
```

## Conclusion

Route 53 failover routing provides DNS-based disaster recovery without application changes. Keep TTL values low (60-300 seconds) on failover records to minimize the time clients continue routing to failed endpoints. Test failover regularly by disabling the primary health check and verifying traffic shifts to the secondary within the expected timeframe. Use `evaluate_target_health = true` on ALB aliases to leverage ALB's own health checking as an additional failover trigger.
