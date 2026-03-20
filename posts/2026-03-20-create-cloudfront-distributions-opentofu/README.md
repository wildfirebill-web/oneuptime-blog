# How to Create CloudFront Distributions with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, AWS, CloudFront, CDN, Terraform, IaC, DevOps

Description: Learn how to create AWS CloudFront distributions with OpenTofu for S3 static sites, ALB origins, custom caching behaviors, and HTTPS with ACM certificates.

## Introduction

CloudFront is AWS's CDN (Content Delivery Network). A distribution sits in front of your origin (S3 bucket or ALB) and serves content from edge locations worldwide. OpenTofu's `aws_cloudfront_distribution` resource covers all CloudFront features including origins, cache behaviors, SSL certificates, and geo-restrictions.

## Prerequisites: ACM Certificate (Must Be in us-east-1)

```hcl
# CloudFront certificates MUST be in us-east-1
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "cdn" {
  provider    = aws.us_east_1
  domain_name = var.domain_name

  subject_alternative_names = ["*.${var.domain_name}"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}
```

## S3 Static Site Distribution

```hcl
resource "aws_s3_bucket" "static_site" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_public_access_block" "static_site" {
  bucket                  = aws_s3_bucket.static_site.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Origin Access Control (OAC) for secure S3 access
resource "aws_cloudfront_origin_access_control" "s3" {
  name                              = "${var.environment}-s3-oac"
  description                       = "OAC for S3 static site"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "s3" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  aliases             = [var.domain_name]
  comment             = "${var.environment} static site CDN"

  origin {
    domain_name              = aws_s3_bucket.static_site.bucket_regional_domain_name
    origin_id                = "s3-${var.bucket_name}"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }

  default_cache_behavior {
    target_origin_id       = "s3-${var.bucket_name}"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true

    cache_policy_id = data.aws_cloudfront_cache_policy.caching_optimized.id
  }

  # SPA: return index.html for 404s
  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cdn.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Name        = "${var.environment}-cdn"
    Environment = var.environment
  }
}
```

## ALB Origin Distribution

```hcl
resource "aws_cloudfront_distribution" "alb" {
  enabled         = true
  is_ipv6_enabled = true
  aliases         = [var.domain_name, "www.${var.domain_name}"]

  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "alb-${var.environment}"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    target_origin_id       = "alb-${var.environment}"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true

    # Use CachingDisabled policy for dynamic content
    cache_policy_id          = data.aws_cloudfront_cache_policy.caching_disabled.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer.id
  }

  # Cache static assets differently
  ordered_cache_behavior {
    path_pattern           = "/static/*"
    target_origin_id       = "alb-${var.environment}"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true
    cache_policy_id        = data.aws_cloudfront_cache_policy.caching_optimized.id
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cdn.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = { Name = "${var.environment}-cdn" }
}
```

## Cache Policy References

```hcl
data "aws_cloudfront_cache_policy" "caching_optimized" {
  name = "Managed-CachingOptimized"
}

data "aws_cloudfront_cache_policy" "caching_disabled" {
  name = "Managed-CachingDisabled"
}

data "aws_cloudfront_origin_request_policy" "all_viewer" {
  name = "Managed-AllViewer"
}
```

## Route53 DNS Record

```hcl
resource "aws_route53_record" "cdn" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.s3.domain_name
    zone_id                = aws_cloudfront_distribution.s3.hosted_zone_id
    evaluate_target_health = false
  }
}
```

## Outputs

```hcl
output "distribution_id"     { value = aws_cloudfront_distribution.s3.id }
output "distribution_arn"    { value = aws_cloudfront_distribution.s3.arn }
output "domain_name"         { value = aws_cloudfront_distribution.s3.domain_name }
output "hosted_zone_id"      { value = aws_cloudfront_distribution.s3.hosted_zone_id }
```

## Conclusion

CloudFront distributions in OpenTofu require an origin (S3 or ALB), cache behaviors, SSL certificate, and viewer restrictions. Always use Origin Access Control (OAC) instead of Origin Access Identity (OAI) for S3 origins. ACM certificates for CloudFront must be created in `us-east-1`. Use managed cache policies (`Managed-CachingOptimized`, `Managed-CachingDisabled`) rather than custom cache settings. For SPAs, add a `custom_error_response` to redirect 404s to `index.html`.
