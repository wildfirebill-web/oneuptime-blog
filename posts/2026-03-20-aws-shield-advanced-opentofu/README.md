# How to Set Up AWS Shield Advanced with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Shield Advanced, DDoS Protection, Route 53, CloudFront, Infrastructure as Code

Description: Learn how to configure AWS Shield Advanced with OpenTofu to protect applications against large DDoS attacks with automatic mitigation, cost protection, and 24/7 DDoS Response Team access.

## Introduction

AWS Shield Advanced provides enhanced DDoS protection beyond the free Shield Standard, including automatic mitigation for large volumetric attacks, near-real-time attack visibility, DDoS cost protection for AWS charges incurred during attacks, and 24/7 access to the AWS DDoS Response Team (DRT). It protects CloudFront, Route 53, ELB, EC2, and Global Accelerator resources.

## Prerequisites

- OpenTofu v1.6+
- AWS Shield Advanced subscription ($3,000/month minimum)
- AWS credentials with Shield and IAM permissions

## Step 1: Enable Shield Advanced Subscription

```hcl
# Enable Shield Advanced for the account

resource "aws_shield_subscription" "main" {
  auto_renew = "ENABLED"  # or "DISABLED"
}
```

## Step 2: Protect Resources

```hcl
# Protect an Application Load Balancer
resource "aws_shield_protection" "alb" {
  name         = "${var.project_name}-alb-protection"
  resource_arn = var.alb_arn

  tags = {
    Name = "${var.project_name}-alb-protection"
  }
}

# Protect a CloudFront distribution
resource "aws_shield_protection" "cloudfront" {
  name         = "${var.project_name}-cloudfront-protection"
  resource_arn = var.cloudfront_distribution_arn

  tags = {
    Name = "${var.project_name}-cloudfront-protection"
  }
}

# Protect a Route 53 hosted zone
resource "aws_shield_protection" "route53" {
  name         = "${var.project_name}-route53-protection"
  resource_arn = "arn:aws:route53:::hostedzone/${var.hosted_zone_id}"

  tags = {
    Name = "${var.project_name}-route53-protection"
  }
}

# Protect an Elastic IP
resource "aws_shield_protection" "eip" {
  name         = "${var.project_name}-eip-protection"
  resource_arn = "arn:aws:ec2:${var.region}:${data.aws_caller_identity.current.account_id}:eip-allocation/${var.eip_allocation_id}"

  tags = {
    Name = "${var.project_name}-eip-protection"
  }
}
```

## Step 3: Create Protection Group for Multiple Resources

```hcl
# Group all protected resources for aggregate attack detection
resource "aws_shield_protection_group" "web_tier" {
  protection_group_id = "${var.project_name}-web-tier"
  aggregation         = "MAX"    # Report the max attack volume across grouped resources
  pattern             = "ARBITRARY"

  members = [
    aws_shield_protection.alb.id,
    aws_shield_protection.cloudfront.id
  ]

  tags = {
    Name = "${var.project_name}-web-tier"
  }
}

# Auto-include all resources of a type
resource "aws_shield_protection_group" "all_cloudfront" {
  protection_group_id = "${var.project_name}-all-cloudfront"
  aggregation         = "SUM"
  pattern             = "BY_RESOURCE_TYPE"
  resource_type       = "CLOUDFRONT_DISTRIBUTION"
}
```

## Step 4: Associate WAF Web ACL for L7 Protection

```hcl
# Shield Advanced requires WAF for L7 DDoS mitigation
resource "aws_wafv2_web_acl_association" "shield" {
  resource_arn = var.alb_arn
  web_acl_arn  = var.waf_web_acl_arn
}

# Shield Advanced Emergency Rate Limiting via WAF
# The DRT can automatically create WAF rules to block attack traffic
resource "aws_shield_drt_access_role_arn_association" "main" {
  role_arn = aws_iam_role.shield_drt.arn
}

resource "aws_iam_role" "shield_drt" {
  name = "${var.project_name}-shield-drt-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "drt.shield.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "shield_drt" {
  role       = aws_iam_role.shield_drt.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSShieldDRTAccessPolicy"
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# View protected resources
aws shield list-protections

# View active attacks
aws shield list-attacks \
  --resource-arns <arn> \
  --start-time '{"FromInclusive": "2026-03-19T00:00:00Z"}'
```

## Conclusion

Shield Advanced is cost-effective for applications generating significant AWS charges that would be impacted by DDoS-related cost spikes, as the DDoS cost protection feature reimburses AWS charges caused by attack traffic. Protect all public-facing resources in a protection group to enable aggregate attack detection, and always associate WAF with protected ALBs to enable Layer 7 automatic mitigation by the DRT during active attacks.
