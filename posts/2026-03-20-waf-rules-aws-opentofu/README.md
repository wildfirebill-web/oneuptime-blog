# How to Configure WAF Rules with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, WAF, Security, Web Application Firewall, CloudFront, ALB

Description: Learn how to create AWS WAF Web ACLs with managed rule groups and custom rules using OpenTofu to protect web applications from OWASP threats and DDoS attacks.

---

AWS WAF protects your web applications from common exploits like SQL injection, XSS, and rate-based attacks. Without WAF, your origins are exposed to malicious traffic. With OpenTofu, you define WAF rules as code, making security policies reviewable and consistent across environments.

## Creating a Web ACL for CloudFront

```hcl
# main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }
}

# WAF for CloudFront must be in us-east-1
provider "aws" {
  region = "us-east-1"
  alias  = "global"
}

# IP set for rate limiting exceptions (e.g., known crawlers)
resource "aws_wafv2_ip_set" "allowed_bots" {
  provider           = aws.global
  name               = "allowed-bots"
  description        = "Trusted bots exempt from rate limiting"
  scope              = "CLOUDFRONT"
  ip_address_version = "IPV4"

  addresses = var.trusted_bot_ips
}

# Web ACL with AWS managed rule groups
resource "aws_wafv2_web_acl" "main" {
  provider    = aws.global
  name        = "${var.project_name}-waf"
  description = "WAF rules for ${var.project_name}"
  scope       = "CLOUDFRONT"  # Use REGIONAL for ALB/API GW

  # Default action: allow all requests not blocked by rules
  default_action {
    allow {}
  }

  # Rule 1: AWS Managed Rules - Core Rule Set (OWASP Top 10)
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 10

    override_action {
      none {}  # Use the managed rule group's action (block/count)
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        # Override specific rules to count rather than block during testing
        rule_action_override {
          name = "SizeRestrictions_BODY"
          action_to_use {
            count {}  # Count but don't block - API payloads can be large
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # Rule 2: Known Bad Inputs
  rule {
    name     = "AWSManagedRulesKnownBadInputsRuleSet"
    priority = 20

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "KnownBadInputs"
      sampled_requests_enabled   = true
    }
  }

  # Rule 3: Rate limiting - 1000 requests per IP per 5 minutes
  rule {
    name     = "RateLimit"
    priority = 30

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 1000
        aggregate_key_type = "IP"

        # Exempt trusted bots from rate limiting
        scope_down_statement {
          not_statement {
            statement {
              ip_set_reference_statement {
                arn = aws_wafv2_ip_set.allowed_bots.arn
              }
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }

  # Rule 4: Custom rule - block specific user agents
  rule {
    name     = "BlockMaliciousUserAgents"
    priority = 40

    action {
      block {}
    }

    statement {
      byte_match_statement {
        field_to_match {
          single_header {
            name = "user-agent"
          }
        }
        positional_constraint = "CONTAINS"
        search_string         = "scanner"
        text_transformation {
          priority = 0
          type     = "LOWERCASE"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "BlockedUserAgents"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.project_name}-waf"
    sampled_requests_enabled   = true
  }

  tags = var.common_tags
}
```

## Attaching WAF to CloudFront

```hcl
# attach.tf
resource "aws_cloudfront_distribution" "main" {
  # ... other configuration ...

  # Attach the WAF Web ACL
  web_acl_id = aws_wafv2_web_acl.main.arn
}
```

## Best Practices

- Start with rule groups in `count` mode rather than `block` mode - analyze logs for false positives before enabling blocking.
- Use AWS Managed Rule Groups for common threats - they're continuously updated by AWS security researchers.
- Set rate limits appropriate for your application - too low blocks legitimate traffic; too high doesn't protect.
- Enable CloudWatch logging for WAF to inspect blocked requests and tune rules.
- Use IP sets for allowlisting trusted sources (monitoring services, payment processors) to prevent false positives.
