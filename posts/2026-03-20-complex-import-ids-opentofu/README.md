# How to Handle Resources with Complex Import IDs in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Import, State Management, AWS, Best Practices, Infrastructure as Code

Description: Learn how to determine and format complex import IDs for AWS resources like IAM policies, DynamoDB tables, and composite-key resources using OpenTofu.

## Introduction

Importing existing resources into OpenTofu state requires knowing the exact format of the resource's import ID. While some resources use simple ARNs or names, others use composite IDs, specific delimiters, or non-obvious formats. This guide covers common complex cases.

## Finding Import ID Formats

Always check the resource documentation for the import format:

```bash
# Check the provider docs or use the OpenTofu registry
# The import section at the bottom of each resource page shows the format
```

## IAM Resources

```bash
# IAM Role Policy Attachment: role_name/policy_arn
tofu import aws_iam_role_policy_attachment.example \
  "my-role/arn:aws:iam::123456789012:policy/MyPolicy"

# IAM User Group Membership: user_name/group_name
tofu import aws_iam_user_group_membership.example \
  "john.doe/developers"

# IAM Policy Document (inline): role_name:policy_name
tofu import aws_iam_role_policy.example \
  "my-role:MyInlinePolicy"
```

## DynamoDB Table

```bash
# DynamoDB table: just the table name
tofu import aws_dynamodb_table.example my-table

# DynamoDB Table Item: table_name/hash_key_value (with range key)
# table_name/hash_key_value/range_key_value
tofu import aws_dynamodb_table_item.example \
  "my-table/HASH_VALUE/RANGE_VALUE"
```

## Route53

```bash
# Route53 Record: zone_id/name/type
tofu import aws_route53_record.example \
  "ZXXXXXXXXXXX/_api.example.com/CNAME"

# Route53 Zone Association: zone_id:vpc_id
tofu import aws_route53_vpc_association_authorization.example \
  "ZXXXXXXXXXXX:vpc-0abc123def456789"
```

## Security Group Rules

```bash
# Security Group Rule: sg_id_rule_type_protocol_from_port_to_port_source
tofu import aws_security_group_rule.example \
  "sgrule-0abc1234_ingress_tcp_443_443_0.0.0.0/0"

# Note: the ID is the sgrule-xxx ID visible in the AWS console
```

## ECS

```bash
# ECS Service: cluster_name/service_name
tofu import aws_ecs_service.example my-cluster/my-service

# ECS Task Definition: task_definition_arn
tofu import aws_ecs_task_definition.example \
  "arn:aws:ecs:us-east-1:123456789012:task-definition/my-task:5"
```

## Using Import Blocks (OpenTofu 1.5+)

```hcl
# imports.tf – loopable import blocks (OpenTofu 1.7+)
locals {
  existing_buckets = {
    "logs"    = "my-company-logs-bucket"
    "backups" = "my-company-backups-bucket"
    "assets"  = "my-company-assets-bucket"
  }
}

import {
  for_each = local.existing_buckets
  id       = each.value
  to       = aws_s3_bucket.imported[each.key]
}

resource "aws_s3_bucket" "imported" {
  for_each = local.existing_buckets
  bucket   = each.value
}
```

## Debugging Import ID Issues

```bash
# If unsure of the ID format, describe the resource from AWS CLI
# and look at identifiers in the response

# For an IAM role policy attachment:
aws iam list-attached-role-policies --role-name my-role

# Import ID format: role-name/policy-arn
tofu import aws_iam_role_policy_attachment.example \
  "my-role/arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"

# Verify the import was successful
tofu plan  # should show no changes
```

## Summary

Complex import IDs in OpenTofu follow provider-specific conventions — composite keys separated by `/`, `:`, or `_`. The resource's documentation import section is the authoritative reference. Using import blocks with `for_each` (OpenTofu 1.7+) simplifies bulk imports. Always run `tofu plan` after importing to verify the state matches the live resource configuration.
