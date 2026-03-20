# How to Create IAM Users and Roles with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, IAM, Users, Roles, Security

Description: Learn how to provision AWS IAM users, groups, and roles with appropriate policies using OpenTofu.

---

IAM users represent humans or applications needing AWS access. IAM roles are assumed by services or users. OpenTofu manages both with consistent, auditable configurations.

---

## Create an IAM User

```hcl
resource "aws_iam_user" "developer" {
  name          = "jane.developer"
  path          = "/users/"
  force_destroy = true

  tags = {
    Team = "platform"
  }
}
```

---

## Create Access Keys for a User

```hcl
resource "aws_iam_access_key" "developer" {
  user = aws_iam_user.developer.name
}

output "access_key_id" {
  value = aws_iam_access_key.developer.id
}

output "secret_access_key" {
  value     = aws_iam_access_key.developer.secret
  sensitive = true
}
```

---

## Create a Group and Add Users

```hcl
resource "aws_iam_group" "developers" {
  name = "developers"
}

resource "aws_iam_group_membership" "developers" {
  name  = "developers-membership"
  group = aws_iam_group.developers.name
  users = [aws_iam_user.developer.name]
}

resource "aws_iam_group_policy_attachment" "developers" {
  group      = aws_iam_group.developers.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

---

## Create an IAM Role

```hcl
resource "aws_iam_role" "lambda_exec" {
  name = "lambda-exec-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = {
    Purpose = "Lambda execution"
  }
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}
```

---

## Cross-Account Role

```hcl
resource "aws_iam_role" "cross_account" {
  name = "cross-account-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::987654321098:root" }
      Action    = "sts:AssumeRole"
      Condition = {
        Bool = { "aws:MultiFactorAuthPresent" = "true" }
      }
    }]
  })
}
```

---

## Summary

Create `aws_iam_user` for humans/applications needing static credentials, `aws_iam_group` for permission grouping, and `aws_iam_role` for service-to-service access. Attach policies with `aws_iam_role_policy_attachment` or `aws_iam_group_policy_attachment`. Use `aws_iam_access_key` to generate credentials and mark outputs as `sensitive = true`.
