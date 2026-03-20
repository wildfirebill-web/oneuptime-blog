# How to Create IAM Policies with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, IAM, Policies, Security, Infrastructure as Code

Description: Learn how to define and attach AWS IAM policies using OpenTofu, including inline policies, managed policies, and policy documents.

---

IAM policies define permissions in AWS. OpenTofu's `aws_iam_policy` and `aws_iam_policy_document` data sources let you declare policies as code, making them reviewable, testable, and versioned.

---

## Using aws_iam_policy_document Data Source

```hcl
data "aws_iam_policy_document" "s3_read" {
  statement {
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:ListBucket",
    ]
    resources = [
      aws_s3_bucket.data.arn,
      "${aws_s3_bucket.data.arn}/*",
    ]
  }
}
```

---

## Create a Managed Policy

```hcl
resource "aws_iam_policy" "s3_read" {
  name        = "S3ReadAccess"
  description = "Read-only access to the data S3 bucket"
  policy      = data.aws_iam_policy_document.s3_read.json
}
```

---

## Attach a Policy to a Role

```hcl
resource "aws_iam_role_policy_attachment" "s3_read" {
  role       = aws_iam_role.app.name
  policy_arn = aws_iam_policy.s3_read.arn
}
```

---

## Inline Policy on a Role

```hcl
resource "aws_iam_role_policy" "inline" {
  name = "inline-policy"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
      Resource = "arn:aws:logs:*:*:*"
    }]
  })
}
```

---

## Policy with Conditions

```hcl
data "aws_iam_policy_document" "restricted_s3" {
  statement {
    effect    = "Allow"
    actions   = ["s3:*"]
    resources = ["*"]
    condition {
      test     = "StringEquals"
      variable = "aws:RequestedRegion"
      values   = ["us-east-1", "us-west-2"]
    }
  }
}
```

---

## Attach AWS Managed Policies

```hcl
resource "aws_iam_role_policy_attachment" "readonly" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

---

## Summary

Use `aws_iam_policy_document` to compose policies with statements, actions, resources, and conditions. Create managed policies with `aws_iam_policy` and attach them with `aws_iam_role_policy_attachment`. Use `aws_iam_role_policy` for inline policies that are specific to one role. Prefer managed policies over inline for reusability and central management.
