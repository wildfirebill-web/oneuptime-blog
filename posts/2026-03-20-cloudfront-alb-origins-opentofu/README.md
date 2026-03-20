# How to Create CloudFront Distributions with ALB Origins in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, CloudFront, ALB, CDN, Infrastructure as Code

Description: Learn how to create CloudFront distributions with Application Load Balancer origins for dynamic content caching and DDoS protection using OpenTofu.

## Introduction

Using CloudFront in front of an Application Load Balancer (ALB) provides global edge caching, DDoS protection via AWS Shield, and SSL termination at the edge. OpenTofu manages the distribution, custom headers for origin validation, and cache behaviors.

## Custom Header for Origin Validation

Restrict the ALB to only accept traffic from CloudFront by requiring a secret header.

```hcl
resource "random_password" "cf_secret" {
  length  = 32
  special = false
}

# Add a listener rule on the ALB to block requests without the header
resource "aws_lb_listener_rule" "cf_only" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 1

  condition {
    http_header {
      http_header_name = "X-CloudFront-Secret"
      values           = [random_password.cf_secret.result]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

## CloudFront Distribution with ALB Origin

```hcl
resource "aws_cloudfront_distribution" "app" {
  enabled         = true
  is_ipv6_enabled = true
  price_class     = "PriceClass_All"

  aliases = [var.app_domain]

  origin {
    domain_name = aws_lb.app.dns_name
    origin_id   = "ALB-${var.app_name}"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }

    # Pass secret header to validate CloudFront origin
    custom_header {
      name  = "X-CloudFront-Secret"
      value = random_password.cf_secret.result
    }
  }

  # Default cache behavior – do not cache dynamic content
  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "ALB-${var.app_name}"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    forwarded_values {
      query_string = true
      headers      = ["Host", "Authorization", "Accept", "Accept-Language"]
      cookies {
        forward = "all"
      }
    }

    min_ttl     = 0
    default_ttl = 0  # no caching for dynamic content
    max_ttl     = 0
  }

  # Cache static assets aggressively
  ordered_cache_behavior {
    path_pattern     = "/static/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "ALB-${var.app_name}"

    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }

    min_ttl     = 0
    default_ttl = 86400    # 1 day
    max_ttl     = 31536000 # 1 year
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    acm_certificate_arn      = var.acm_certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## WAF Web ACL

```hcl
resource "aws_wafv2_web_acl_association" "cloudfront" {
  resource_arn = aws_cloudfront_distribution.app.arn
  web_acl_arn  = aws_wafv2_web_acl.cf.arn
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

CloudFront with an ALB origin provides edge caching, DDoS protection, and SSL termination for dynamic applications. OpenTofu manages the distribution, origin validation headers, cache behaviors for static and dynamic content, and WAF associations — creating a secure, globally distributed application delivery layer.
