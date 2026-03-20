# How to Handle Deprecated Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Deprecated Resources, Migration, Provider, Best Practices, Infrastructure as Code

Description: Learn how to identify, migrate from, and safely replace deprecated OpenTofu provider resources without causing infrastructure disruptions.

## Introduction

Provider resources are deprecated when cloud services introduce better replacements. Deprecated resources continue to work but receive no new features and may be removed in future provider versions. This guide covers how to identify deprecated resources and migrate safely.

## Identifying Deprecated Resources

```bash
# Run validate to surface deprecation warnings

tofu validate

# Run plan with upgrade flag to see migration hints
tofu providers lock -upgrade

# Check the provider changelog for removed resources
# Example for AWS provider:
# https://github.com/hashicorp/terraform-provider-aws/blob/main/CHANGELOG.md
```

## Example: CloudFront OAI to OAC Migration

`aws_cloudfront_origin_access_identity` is still functional but OAC is recommended.

```hcl
# Old approach (OAI) – still works but considered legacy
resource "aws_cloudfront_origin_access_identity" "old" {
  comment = "Legacy OAI"
}

# Step 1: Add new OAC alongside the old OAI
resource "aws_cloudfront_origin_access_control" "new" {
  name                              = "${var.app_name}-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Step 2: Update the distribution to use OAC
resource "aws_cloudfront_distribution" "main" {
  origin {
    domain_name              = aws_s3_bucket.main.bucket_regional_domain_name
    origin_id                = "S3"
    # Switch from OAI to OAC
    origin_access_control_id = aws_cloudfront_origin_access_control.new.id
    # Remove: s3_origin_config block
  }
  # ...
}

# Step 3: Update bucket policy to use OAC-style principal
resource "aws_s3_bucket_policy" "main" {
  bucket = aws_s3_bucket.main.id
  policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.main.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.main.arn
        }
      }
    }]
  })
}

# Step 4: After verifying OAC works, remove old OAI resource
# tofu state rm aws_cloudfront_origin_access_identity.old
# Then delete the resource block from code
```

## Example: AWS WAF Classic to WAFv2

```hcl
# Old: WAF Classic (aws_waf_*)
# resource "aws_waf_web_acl" "classic" { ... }

# New: WAFv2
resource "aws_wafv2_web_acl" "main" {
  name  = "${var.app_name}-waf"
  scope = "CLOUDFRONT"

  default_action {
    allow {}
  }

  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action { none {} }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "${var.app_name}-waf"
    sampled_requests_enabled   = true
  }
}
```

## Safe Deprecation Migration Steps

```bash
# 1. Add the replacement resource to your code
# 2. Plan to verify it creates correctly
tofu plan

# 3. Apply to create the new resource
tofu apply -target=aws_wafv2_web_acl.main

# 4. Update references to point to the new resource
# 5. Apply to update downstream resources
tofu apply

# 6. Remove the old resource from code
# 7. If needed, remove from state without destroying
tofu state rm aws_waf_web_acl.classic

# 8. Optionally manually destroy the old resource
aws wafv2 delete-web-acl --id ... --scope REGIONAL --lock-token ...
```

## Summary

Handling deprecated resources requires a phased migration: add the replacement resource alongside the old one, verify it works, update all references, then safely remove the deprecated resource from both code and state. Always test in non-production first and use `tofu state rm` to decouple state removal from resource destruction.
