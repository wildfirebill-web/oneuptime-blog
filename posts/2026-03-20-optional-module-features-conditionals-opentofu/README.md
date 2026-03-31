# How to Handle Optional Module Features with Conditionals in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Module, Conditional, Optional, HCL, Design Pattern

Description: Learn how to design OpenTofu modules with optional feature flags that let callers enable or disable specific capabilities without changing module code.

## Introduction

Well-designed modules expose feature flags that let callers opt into optional capabilities - WAF protection, enhanced monitoring, read replicas, CDN, etc. This guide shows patterns for building modules that gracefully handle optional features.

## Feature Flags Variable Pattern

```hcl
# modules/web-app/variables.tf

variable "features" {
  description = "Optional features to enable for the web application"
  type = object({
    enable_cdn             = optional(bool, false)
    enable_waf             = optional(bool, false)
    enable_shield          = optional(bool, false)
    enable_read_replicas   = optional(bool, false)
    read_replica_count     = optional(number, 1)
    enable_enhanced_monitoring = optional(bool, false)
    enable_performance_insights = optional(bool, false)
  })
  default = {}
}
```

```hcl
# modules/web-app/main.tf
resource "aws_wafv2_web_acl_association" "app" {
  count        = var.features.enable_waf ? 1 : 0
  resource_arn = aws_lb.app.arn
  web_acl_arn  = aws_wafv2_web_acl.app[0].arn
}

resource "aws_wafv2_web_acl" "app" {
  count = var.features.enable_waf ? 1 : 0
  name  = "${var.app_name}-waf"
  scope = "REGIONAL"

  default_action { allow {} }
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.app_name}-waf"
    sampled_requests_enabled   = true
  }
}

resource "aws_db_instance" "read_replica" {
  count               = var.features.enable_read_replicas ? var.features.read_replica_count : 0
  replicate_source_db = aws_db_instance.primary.id
  instance_class      = var.db_instance_class
  publicly_accessible = false

  performance_insights_enabled = var.features.enable_performance_insights
  monitoring_interval          = var.features.enable_enhanced_monitoring ? 60 : 0
}
```

## Caller Usage: Simple and Advanced

```hcl
# Simple deployment - all features use defaults (disabled)
module "web_app_dev" {
  source   = "./modules/web-app"
  app_name = "myapp"
  environment = "dev"
  # features = {} -- uses all defaults
}

# Production deployment with specific features enabled
module "web_app_prod" {
  source      = "./modules/web-app"
  app_name    = "myapp"
  environment = "prod"

  features = {
    enable_cdn                  = true
    enable_waf                  = true
    enable_read_replicas        = true
    read_replica_count          = 2
    enable_enhanced_monitoring  = true
    enable_performance_insights = true
  }
}
```

## Conditional Outputs from Optional Features

Module outputs should gracefully handle disabled features.

```hcl
# modules/web-app/outputs.tf
output "cdn_domain" {
  description = "CloudFront domain name, or null if CDN is disabled"
  value       = var.features.enable_cdn ? aws_cloudfront_distribution.app[0].domain_name : null
}

output "read_replica_endpoints" {
  description = "Read replica endpoints, or empty list if disabled"
  value = var.features.enable_read_replicas ? [
    for replica in aws_db_instance.read_replica : replica.endpoint
  ] : []
}

output "waf_arn" {
  description = "WAF ACL ARN, or null if WAF is disabled"
  value       = var.features.enable_waf ? aws_wafv2_web_acl.app[0].arn : null
}
```

## Feature-Gated Sub-Modules

For large feature sets, encapsulate in sub-modules.

```hcl
module "observability" {
  source  = "./modules/observability"
  enabled = var.features.enable_enhanced_monitoring

  app_name     = var.app_name
  alarm_topic  = var.alarm_sns_topic_arn
}

module "cdn" {
  source  = "./modules/cdn"
  enabled = var.features.enable_cdn

  origin_domain    = aws_lb.app.dns_name
  certificate_arn  = var.features.enable_cdn ? var.certificate_arn : ""
  custom_domain    = var.custom_domain
}
```

## Conclusion

Feature flag patterns in modules give callers fine-grained control over which capabilities are provisioned, making modules reusable across environments with different requirements. The `optional()` type constraint with defaults makes calling modules ergonomic - callers only specify what they want to change from the defaults.
