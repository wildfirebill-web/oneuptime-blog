# How to Create IAM Policies with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, IAM Policies, Security, Infrastructure as Code

Description: Learn how to create and manage AWS IAM policies with OpenTofu using the aws_iam_policy_document data source for readable, version-controlled access control.

## Introduction

IAM policies define what actions are allowed or denied on which AWS resources. OpenTofu's `aws_iam_policy_document` data source makes writing policies readable and type-safe-far easier to maintain than raw JSON strings.

## Using `aws_iam_policy_document`

```hcl
# Define the policy document using the data source

data "aws_iam_policy_document" "s3_read_write" {
  statement {
    sid    = "ListBucket"
    effect = "Allow"

    actions   = ["s3:ListBucket", "s3:GetBucketLocation"]
    resources = [aws_s3_bucket.data.arn]
  }

  statement {
    sid    = "ReadWriteObjects"
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
    ]
    # Allow access to all objects in the bucket
    resources = ["${aws_s3_bucket.data.arn}/*"]
  }
}

# Create the managed policy from the document
resource "aws_iam_policy" "s3_read_write" {
  name        = "s3-data-read-write"
  description = "Read and write access to the data S3 bucket"
  policy      = data.aws_iam_policy_document.s3_read_write.json
}
```

## Inline vs Managed Policies

```hcl
# Inline policy: attached directly to a role, not reusable
resource "aws_iam_role_policy" "lambda_inline" {
  name   = "lambda-s3-access"
  role   = aws_iam_role.lambda.id
  policy = data.aws_iam_policy_document.s3_read_write.json
}

# Managed policy: standalone, can be attached to multiple roles/users
resource "aws_iam_policy" "s3_managed" {
  name   = "s3-data-access"
  policy = data.aws_iam_policy_document.s3_read_write.json
}

resource "aws_iam_role_policy_attachment" "lambda_s3" {
  role       = aws_iam_role.lambda.name
  policy_arn = aws_iam_policy.s3_managed.arn
}
```

## Condition Keys

Add conditions to restrict permissions:

```hcl
data "aws_iam_policy_document" "region_restricted" {
  statement {
    effect    = "Allow"
    actions   = ["ec2:*"]
    resources = ["*"]

    condition {
      test     = "StringEquals"
      variable = "aws:RequestedRegion"
      values   = ["us-east-1", "us-west-2"]
    }

    condition {
      test     = "StringEquals"
      variable = "aws:PrincipalTag/Department"
      values   = ["platform"]
    }
  }

  statement {
    # Deny access if MFA is not present (for human users)
    effect    = "Deny"
    actions   = ["*"]
    resources = ["*"]

    condition {
      test     = "BoolIfExists"
      variable = "aws:MultiFactorAuthPresent"
      values   = ["false"]
    }
  }
}
```

Resource-Based Policies

Some resources (S3, SQS, SNS, KMS) also support resource-based policies:

```hcl
data "aws_iam_policy_document" "sqs_allow_sns" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["sns.amazonaws.com"]
    }

    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.main.arn]

    condition {
      test     = "ArnEquals"
      variable = "aws:SourceArn"
      values   = [aws_sns_topic.events.arn]
    }
  }
}

resource "aws_sqs_queue_policy" "main" {
  queue_url = aws_sqs_queue.main.id
  policy    = data.aws_iam_policy_document.sqs_allow_sns.json
}
```

## Permission Boundary

```hcl
# A permission boundary limits the maximum permissions a role can have
data "aws_iam_policy_document" "developer_boundary" {
  statement {
    effect    = "Allow"
    actions   = ["s3:*", "dynamodb:*", "lambda:*"]
    resources = ["*"]
  }

  statement {
    effect    = "Deny"
    actions   = ["iam:*"]
    resources = ["*"]
  }
}

resource "aws_iam_policy" "developer_boundary" {
  name   = "DeveloperPermissionBoundary"
  policy = data.aws_iam_policy_document.developer_boundary.json
}

resource "aws_iam_role" "developer_role" {
  name                  = "developer-role"
  assume_role_policy    = data.aws_iam_policy_document.ec2_trust.json
  permissions_boundary  = aws_iam_policy.developer_boundary.arn
}
```

## Conclusion

The `aws_iam_policy_document` data source transforms IAM policy authoring from error-prone JSON editing to structured, readable HCL. Use managed policies for reusability, add conditions for fine-grained control, and apply permission boundaries when delegating role creation to other teams.
