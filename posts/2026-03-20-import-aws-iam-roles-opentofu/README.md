# How to Import AWS IAM Roles into OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, AWS, IAM, Import, Security

Description: Learn how to import existing AWS IAM roles, policies, and policy attachments into OpenTofu state to bring manually created IAM resources under infrastructure-as-code management.

## Introduction

IAM roles created manually or by other tools can be imported into OpenTofu. The challenge is that IAM roles have multiple associated resources — assume role policies, inline policies, and managed policy attachments — that must all be imported separately.

## Step 1: Inventory the IAM Role

```bash
ROLE_NAME="my-app-role"

# Get assume role policy
aws iam get-role \
  --role-name $ROLE_NAME \
  --query 'Role.AssumeRolePolicyDocument' \
  --output json | python3 -m json.tool

# List attached managed policies
aws iam list-attached-role-policies \
  --role-name $ROLE_NAME \
  --query 'AttachedPolicies[].[PolicyArn,PolicyName]' \
  --output table

# List inline policies
aws iam list-role-policies \
  --role-name $ROLE_NAME \
  --query 'PolicyNames' \
  --output table

# Get inline policy content
aws iam get-role-policy \
  --role-name $ROLE_NAME \
  --policy-name "InlinePolicyName"
```

## Step 2: Write Matching HCL

```hcl
data "aws_iam_policy_document" "assume_role" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "app" {
  name               = "my-app-role"
  description        = "Role for application EC2 instances"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
  path               = "/"

  tags = {
    Environment = "prod"
    ManagedBy   = "OpenTofu"
  }
}

# Managed policy attachments
resource "aws_iam_role_policy_attachment" "s3_read" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Inline policy
data "aws_iam_policy_document" "custom" {
  statement {
    effect    = "Allow"
    actions   = ["secretsmanager:GetSecretValue"]
    resources = ["arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/*"]
  }
}

resource "aws_iam_role_policy" "custom" {
  name   = "custom-secrets-access"
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.custom.json
}
```

## Step 3: Import Blocks

```hcl
# import.tf
import {
  to = aws_iam_role.app
  id = "my-app-role"
}

# Policy attachments use composite key: ROLE_NAME/POLICY_ARN
import {
  to = aws_iam_role_policy_attachment.s3_read
  id = "my-app-role/arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

import {
  to = aws_iam_role_policy_attachment.ssm
  id = "my-app-role/arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Inline policies use composite key: ROLE_NAME:POLICY_NAME
import {
  to = aws_iam_role_policy.custom
  id = "my-app-role:custom-secrets-access"
}
```

## Importing Custom IAM Policies

```hcl
resource "aws_iam_policy" "custom" {
  name   = "MyCustomPolicy"
  path   = "/"
  policy = data.aws_iam_policy_document.custom.json
}

# Import using the policy ARN
import {
  to = aws_iam_policy.custom
  id = "arn:aws:iam::123456789012:policy/MyCustomPolicy"
}
```

## Conclusion

IAM role import requires importing the role plus every policy attachment separately. The composite key format (ROLE/POLICY_ARN for managed attachments, ROLE:POLICY_NAME for inline policies) is non-obvious. Always verify that your HCL's `assume_role_policy` JSON matches exactly what's in AWS — policy JSON comparison is order-sensitive in some cases and may cause spurious plan diffs.
