# How to Create WAF Rate-Based Rules with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, WAF, Rate Limiting, Infrastructure as Code

Description: Learn how to create AWS WAFv2 rate-based rules with OpenTofu to protect against DDoS attacks and API abuse by limiting request rates.

Rate-based WAF rules automatically block IP addresses that exceed a request threshold within a rolling 5-minute window. They're essential for DDoS mitigation and preventing brute-force and credential stuffing attacks.

## Basic Rate-Based Rule

```hcl
resource "aws_wafv2_web_acl" "with_rate_limits" {
  name  = "rate-limited-web-acl"
  scope = "REGIONAL"

  default_action {
    allow {}
  }

  # Rate limit all traffic: block IPs exceeding 2000 requests per 5 min
  rule {
    name     = "GlobalRateLimit"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 2000  # Requests per 5-minute window
        aggregate_key_type = "IP"  # Aggregate by source IP
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "GlobalRateLimit"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "rate-limited-web-acl"
    sampled_requests_enabled   = true
  }
}
```

## Rate Limit Login Endpoint

```hcl
resource "aws_wafv2_web_acl" "api_protected" {
  name  = "api-protected-acl"
  scope = "REGIONAL"

  default_action { allow {} }

  # Strict rate limit on login endpoint to prevent brute force
  rule {
    name     = "LoginRateLimit"
    priority = 1

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 100  # 100 login attempts per 5 min per IP
        aggregate_key_type = "IP"

        # Only apply to /auth/login
        scope_down_statement {
          byte_match_statement {
            field_to_match {
              uri_path {}
            }
            positional_constraint = "STARTS_WITH"
            search_string         = "/auth/login"
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "LoginRateLimit"
      sampled_requests_enabled   = true
    }
  }

  # API rate limit per IP
  rule {
    name     = "APIRateLimit"
    priority = 2

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 5000  # 5000 API requests per 5 min per IP
        aggregate_key_type = "IP"

        scope_down_statement {
          byte_match_statement {
            field_to_match { uri_path {} }
            positional_constraint = "STARTS_WITH"
            search_string         = "/api/"
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "APIRateLimit"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "api-protected-acl"
    sampled_requests_enabled   = true
  }
}
```

## Forwarded IP Rate Limiting (Behind Load Balancer)

```hcl
rule {
  name     = "ForwardedIPRateLimit"
  priority = 3

  action { block {} }

  statement {
    rate_based_statement {
      limit              = 3000
      aggregate_key_type = "FORWARDED_IP"

      forwarded_ip_config {
        header_name       = "X-Forwarded-For"
        fallback_behavior = "MATCH"  # Block if header missing
      }
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "ForwardedIPRateLimit"
    sampled_requests_enabled   = true
  }
}
```

## Rate Limit Monitoring

```hcl
resource "aws_cloudwatch_metric_alarm" "high_block_rate" {
  alarm_name          = "waf-high-rate-limit-blocks"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "BlockedRequests"
  namespace           = "AWS/WAFV2"
  period              = 300
  statistic           = "Sum"
  threshold           = 1000
  alarm_description   = "High number of WAF rate-limit blocks - potential DDoS"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]

  dimensions = {
    WebACL = aws_wafv2_web_acl.api_protected.name
    Region = var.region
    Rule   = "LoginRateLimit"
  }
}
```

## Conclusion

WAFv2 rate-based rules in OpenTofu provide automatic DDoS and brute-force protection. Apply aggressive rate limits to authentication endpoints (100 per 5 minutes), moderate limits to API endpoints (5000 per 5 minutes), and global limits as a backstop. Use scope_down_statement to target specific paths, and monitor BlockedRequests metrics to tune thresholds and detect attack patterns.
