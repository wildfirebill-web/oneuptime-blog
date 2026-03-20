# How to Create AWS WAFv2 Web ACLs with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, WAF, Web Security, Infrastructure as Code

Description: Learn how to create AWS WAFv2 Web ACLs with OpenTofu to protect your applications from common web exploits, SQL injection, and DDoS attacks.

AWS WAFv2 Web ACLs define the rules that filter HTTP requests to your CloudFront, ALB, API Gateway, or AppSync resources. Managing them in OpenTofu ensures consistent security rules are applied across environments.

## Creating a Web ACL

```hcl
resource "aws_wafv2_web_acl" "main" {
  name        = "myapp-web-acl"
  description = "Web ACL for myapp production"
  scope       = "REGIONAL"  # REGIONAL for ALB/API GW, CLOUDFRONT for CloudFront

  default_action {
    allow {}  # Allow by default, rules will block specific traffic
  }

  # AWS Managed Rules — Core Rule Set
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 10

    override_action {
      none {}  # Don't override — use the rule's own action
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "AWSManagedRulesCommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # AWS Managed Rules — Known Bad Inputs
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
      metric_name                = "AWSManagedRulesKnownBadInputs"
      sampled_requests_enabled   = true
    }
  }

  # Block requests from specific countries
  rule {
    name     = "BlockHighRiskCountries"
    priority = 30

    action {
      block {}
    }

    statement {
      geo_match_statement {
        country_codes = ["CN", "RU", "KP"]  # Adjust for your risk profile
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "BlockedCountries"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "myapp-web-acl"
    sampled_requests_enabled   = true
  }

  tags = {
    Environment = "production"
    Team        = "security"
  }
}
```

## Associate Web ACL with ALB

```hcl
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

## Enable Logging

```hcl
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  log_destination_configs = [aws_cloudwatch_log_group.waf.arn]
  resource_arn            = aws_wafv2_web_acl.main.arn

  # Filter out non-blocked requests to reduce log volume
  logging_filter {
    default_behavior = "DROP"  # Don't log unless the condition is met

    filter {
      behavior = "KEEP"

      condition {
        action_condition {
          action = "BLOCK"  # Only log blocked requests
        }
      }

      requirement = "MEETS_ANY"
    }
  }
}

resource "aws_cloudwatch_log_group" "waf" {
  name              = "aws-waf-logs-myapp"  # Must start with aws-waf-logs-
  retention_in_days = 90
}
```

## Conclusion

AWS WAFv2 Web ACLs in OpenTofu provide layered protection against web attacks. Start with AWS Managed Rule Sets for CRS and known bad inputs, add geographic restrictions if needed, and enable logging on blocked requests for security analysis. Associate the ACL with your ALB or CloudFront distribution to activate protection.
