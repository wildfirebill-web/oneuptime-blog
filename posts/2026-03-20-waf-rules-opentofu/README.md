# How to Set Up Web Application Firewall Rules with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WAF, Security, OpenTofu, AWS WAFv2, Azure WAF, Cloud Armor, OWASP

Description: Learn how to configure Web Application Firewall rules across AWS, Azure, and GCP using OpenTofu to protect web applications from OWASP Top 10 attacks.

## Overview

WAF rules protect web applications from SQL injection, XSS, and other attacks. OpenTofu configures managed rule groups and custom rules across AWS WAFv2, Azure Application Gateway WAF, and GCP Cloud Armor.

## Step 1: AWS WAFv2 Web ACL

```hcl
# main.tf - AWS WAFv2 with managed rule groups

resource "aws_wafv2_web_acl" "app" {
  name  = "app-waf"
  scope = "REGIONAL"  # or CLOUDFRONT for edge

  default_action {
    allow {}
  }

  # AWS Managed Rules - Core rule set (OWASP)
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action { none {} }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesCommonRuleSet"

        # Override specific rules to count instead of block
        rule_action_override {
          name          = "SizeRestrictions_BODY"
          action_to_use { count {} }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # SQL injection protection
  rule {
    name     = "AWSManagedRulesSQLiRuleSet"
    priority = 2

    override_action { none {} }

    statement {
      managed_rule_group_statement {
        vendor_name = "AWS"
        name        = "AWSManagedRulesSQLiRuleSet"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "SQLiRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # Custom rate limiting rule
  rule {
    name     = "RateLimitRule"
    priority = 10

    action { block {} }

    statement {
      rate_based_statement {
        limit              = 2000  # 2000 requests per 5 minutes per IP
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateLimit"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "AppWAF"
    sampled_requests_enabled   = true
  }
}

# Associate WAF with ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.app.arn
  web_acl_arn  = aws_wafv2_web_acl.app.arn
}
```

## Step 2: GCP Cloud Armor

```hcl
# GCP Cloud Armor security policy
resource "google_compute_security_policy" "waf" {
  name = "app-waf-policy"

  # Pre-configured WAF rules (OWASP)
  rule {
    action   = "deny(403)"
    priority = 1000
    match {
      expr {
        expression = "evaluatePreconfiguredWaf('sqli-v33-stable', {'sensitivity': 1})"
      }
    }
    description = "Block SQL injection"
  }

  rule {
    action   = "deny(403)"
    priority = 1001
    match {
      expr {
        expression = "evaluatePreconfiguredWaf('xss-v33-stable', {'sensitivity': 1})"
      }
    }
    description = "Block XSS"
  }

  # Rate limiting
  rule {
    action   = "throttle"
    priority = 2000

    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }

    rate_limit_options {
      conform_action = "allow"
      exceed_action  = "deny(429)"
      enforce_on_key = "IP"

      rate_limit_threshold {
        count        = 100
        interval_sec = 60
      }
    }
  }

  # Default allow
  rule {
    action   = "allow"
    priority = 2147483647

    match {
      versioned_expr = "SRC_IPS_V1"
      config {
        src_ip_ranges = ["*"]
      }
    }
  }
}
```

## Summary

WAF rules configured with OpenTofu provide consistent application security across cloud providers. AWS Managed Rule Groups cover OWASP Top 10 with regularly updated threat intelligence, while custom rate limiting rules prevent DDoS and credential stuffing. GCP Cloud Armor's preconfigured WAF rules (using `evaluatePreconfiguredWaf`) provide similar protection with sensitivity controls to balance security against false positives.
