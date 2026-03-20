# How to Set Up Multi-Region Load Balancing with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Load Balancing, Multi-Region, High Availability

Description: Learn how to deploy multi-region load balancing on AWS using OpenTofu, combining Route 53 latency routing with Application Load Balancers for global traffic distribution.

## Introduction

Multi-region load balancing distributes traffic across AWS regions based on latency, geolocation, or failover rules. Using OpenTofu to manage this setup ensures consistent configuration across regions and simplifies changes to routing policies.

## Architecture

This setup uses:
- **Route 53** with latency-based routing to direct users to the nearest region
- **Application Load Balancers** in each region
- **Auto Scaling Groups** behind each ALB
- **Health checks** for automatic failover

## Provider Configuration

```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_southeast"
  region = "ap-southeast-1"
}
```

## Reusable Region Module

Create a module for per-region infrastructure:

```hcl
# modules/region/main.tf

variable "region" {}
variable "vpc_cidr" {}
variable "app_image" {}
variable "certificate_arn" {}

resource "aws_lb" "app" {
  name               = "app-alb-${var.region}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

output "alb_dns_name" { value = aws_lb.app.dns_name }
output "alb_zone_id"  { value = aws_lb.app.zone_id }
```

## Root Configuration

```hcl
# main.tf

module "us_east" {
  source          = "./modules/region"
  region          = "us-east-1"
  vpc_cidr        = "10.0.0.0/16"
  app_image       = var.app_image
  certificate_arn = var.cert_us_east
  providers = { aws = aws.us_east }
}

module "eu_west" {
  source          = "./modules/region"
  region          = "eu-west-1"
  vpc_cidr        = "10.1.0.0/16"
  app_image       = var.app_image
  certificate_arn = var.cert_eu_west
  providers = { aws = aws.eu_west }
}

module "ap_southeast" {
  source          = "./modules/region"
  region          = "ap-southeast-1"
  vpc_cidr        = "10.2.0.0/16"
  app_image       = var.app_image
  certificate_arn = var.cert_ap_southeast
  providers = { aws = aws.ap_southeast }
}
```

## Route 53 Latency Routing

```hcl
resource "aws_route53_zone" "primary" {
  name = "example.com"
}

resource "aws_route53_record" "app_us" {
  zone_id        = aws_route53_zone.primary.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "us-east-1"

  latency_routing_policy {
    region = "us-east-1"
  }

  alias {
    name                   = module.us_east.alb_dns_name
    zone_id                = module.us_east.alb_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_eu" {
  zone_id        = aws_route53_zone.primary.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "eu-west-1"

  latency_routing_policy {
    region = "eu-west-1"
  }

  alias {
    name                   = module.eu_west.alb_dns_name
    zone_id                = module.eu_west.alb_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_ap" {
  zone_id        = aws_route53_zone.primary.zone_id
  name           = "app.example.com"
  type           = "A"
  set_identifier = "ap-southeast-1"

  latency_routing_policy {
    region = "ap-southeast-1"
  }

  alias {
    name                   = module.ap_southeast.alb_dns_name
    zone_id                = module.ap_southeast.alb_zone_id
    evaluate_target_health = true
  }
}
```

## Health Checks for Automatic Failover

```hcl
resource "aws_route53_health_check" "us_east" {
  fqdn              = module.us_east.alb_dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30

  tags = { Name = "us-east-health-check" }
}
```

Apply the health check to the latency record for automatic failover when a region becomes unhealthy.

## Deployment

```bash
tofu init
tofu workspace new production
tofu plan -var-file=production.tfvars
tofu apply -var-file=production.tfvars
```

## Best Practices

- Use ACM certificates in each region separately (ACM certificates are regional).
- Enable access logging on ALBs in each region.
- Use the same container image tag across all regions to ensure consistent behavior.
- Configure cross-region state references carefully to avoid circular dependencies.
- Test region failover manually by temporarily disabling one region's health check.

## Conclusion

OpenTofu makes multi-region load balancing manageable through provider aliases and reusable modules. By combining ALBs with Route 53 latency routing, you achieve global traffic distribution with automatic failover — all defined and deployed as reproducible infrastructure code.
