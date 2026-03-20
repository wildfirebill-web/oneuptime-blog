# How to Set Up AWS Firewall Manager with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Firewall Manager, WAF, Security Groups, Organizations, Infrastructure as Code

Description: Learn how to configure AWS Firewall Manager with OpenTofu to centrally manage WAF rules, security groups, and Shield Advanced protections across all accounts in an AWS Organization.

## Introduction

AWS Firewall Manager provides centralized security policy management across accounts in AWS Organizations. A single policy can enforce WAF Web ACLs, security group rules, Shield Advanced protections, and Network Firewall configurations across hundreds of accounts and resources—new accounts automatically receive the policy when they join the organization.

## Prerequisites

- OpenTofu v1.6+
- AWS Organizations with Firewall Manager admin account designated
- AWS Shield Advanced subscription (for Shield policies)
- AWS credentials with Firewall Manager and Organizations permissions

## Step 1: Designate Firewall Manager Admin Account

```hcl
# Designate an account as Firewall Manager admin (done once in the management account)
resource "aws_fms_admin_account" "main" {
  account_id = var.security_account_id  # Security/audit account
}
```

## Step 2: Create WAF Policy to Enforce Managed Rules

```hcl
# Enforce WAF rules across all ALBs in the Organization
resource "aws_fms_policy" "waf_policy" {
  depends_on = [aws_fms_admin_account.main]

  name                        = "${var.project_name}-waf-policy"
  exclude_resource_tags       = false
  remediation_enabled         = true  # Auto-create/update WAF associations
  resource_type               = "AWS::ElasticLoadBalancingV2::LoadBalancer"

  security_service_policy_data {
    type = "WAFV2"

    managed_service_data = jsonencode({
      type = "WAFV2"
      preProcessRuleGroups = [
        {
          ruleGroupType  = "ManagedRuleGroup"
          managedRuleGroupIdentifier = {
            versionEnabled = false
            vendorName     = "AWS"
            managedRuleGroupName = "AWSManagedRulesCommonRuleSet"
          }
          overrideAction = { type = "NONE" }
          priority       = 1
        }
      ]
      postProcessRuleGroups = []
      defaultAction         = { type = "ALLOW" }
      overrideCustomerWebACLAssociation = false
    })
  }

  include_map {
    account = var.included_account_ids  # Or leave empty to apply to all accounts
  }

  tags = {
    Name = "${var.project_name}-waf-policy"
  }
}
```

## Step 3: Create Security Group Policy

```hcl
# Enforce that all EC2 instances cannot have port 22 (SSH) open to 0.0.0.0/0
resource "aws_fms_policy" "sg_policy" {
  depends_on = [aws_fms_admin_account.main]

  name                  = "${var.project_name}-no-public-ssh"
  remediation_enabled   = true
  resource_type         = "AWS::EC2::SecurityGroup"

  security_service_policy_data {
    type = "SECURITY_GROUPS_USAGE_AUDIT"

    managed_service_data = jsonencode({
      type = "SECURITY_GROUPS_USAGE_AUDIT"
      deleteUnusedSecurityGroups = false
      coalesceRedundantSecurityGroups = true
    })
  }

  tags = {
    Name = "${var.project_name}-no-public-ssh"
  }
}
```

## Step 4: Create Shield Advanced Policy

```hcl
# Enforce Shield Advanced protection on all CloudFront distributions
resource "aws_fms_policy" "shield_cloudfront" {
  depends_on = [aws_fms_admin_account.main]

  name                  = "${var.project_name}-shield-cloudfront"
  remediation_enabled   = true
  resource_type         = "AWS::CloudFront::Distribution"

  security_service_policy_data {
    type = "SHIELD_ADVANCED"
  }

  include_map {
    orgunit = var.production_ou_ids  # Only apply to production OU
  }

  tags = {
    Name = "${var.project_name}-shield-cloudfront"
  }
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# View policy compliance across accounts
aws fms list-compliance-status \
  --policy-id <policy-id>
```

## Conclusion

Firewall Manager is the most efficient tool for security governance at scale in AWS Organizations—a single policy definition enforces security controls across dozens or hundreds of accounts without per-account configuration. Enable `remediation_enabled = true` to automatically apply policies to resources that should be protected but aren't, ensuring new accounts and new resources are automatically covered. Use OUs in `include_map` to apply different policies to production vs. development environments.
