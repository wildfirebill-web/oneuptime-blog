# How to Create IAM Users and Roles with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, IAM, Users, Roles, Security

Description: Learn how to create AWS IAM users, groups, and roles with OpenTofu for managing human and machine identities with least-privilege permissions.

## Introduction

IAM users and roles control access to AWS resources. Users represent human identities; roles represent machine identities (EC2 instances, Lambda functions, CI/CD pipelines). OpenTofu makes both easy to create, configure, and audit as code.

## Creating an IAM User

```hcl
resource "aws_iam_user" "developer" {
  name = "jane.developer"
  path = "/developers/"

  tags = {
    Team        = "platform"
    Environment = "all"
  }
}

# Create an access key for programmatic access

resource "aws_iam_access_key" "developer" {
  user = aws_iam_user.developer.name
}

# Store the secret in Secrets Manager (never output it plaintext)
resource "aws_secretsmanager_secret_version" "developer_key" {
  secret_id = aws_secretsmanager_secret.developer_key.id
  secret_string = jsonencode({
    access_key_id     = aws_iam_access_key.developer.id
    secret_access_key = aws_iam_access_key.developer.secret
  })
}
```

## Creating an IAM Group with Policy

```hcl
resource "aws_iam_group" "developers" {
  name = "developers"
  path = "/groups/"
}

resource "aws_iam_group_policy_attachment" "developers_s3_read" {
  group      = aws_iam_group.developers.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

# Add the user to the group
resource "aws_iam_user_group_membership" "developer" {
  user   = aws_iam_user.developer.name
  groups = [aws_iam_group.developers.name]
}
```

## Creating an IAM Role for EC2

```hcl
resource "aws_iam_role" "ec2_app" {
  name        = "ec2-app-role"
  description = "Role for application EC2 instances"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = { Name = "ec2-app-role" }
}

resource "aws_iam_instance_profile" "ec2_app" {
  name = "ec2-app-profile"
  role = aws_iam_role.ec2_app.name
}
```

## Creating a Cross-Account Role

```hcl
variable "trusted_account_id" {
  description = "AWS account ID allowed to assume this role"
  type        = string
}

resource "aws_iam_role" "cross_account_read" {
  name = "CrossAccountReadRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${var.trusted_account_id}:root"
      }
      Action = "sts:AssumeRole"
      Condition = {
        BoolIfExists = {
          "aws:MultiFactorAuthPresent" = "true"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "cross_account_read" {
  role       = aws_iam_role.cross_account_read.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

## Creating Multiple Roles from a Variable

```hcl
variable "service_roles" {
  type = map(object({
    description  = string
    policy_arns  = list(string)
  }))
}

resource "aws_iam_role" "services" {
  for_each    = var.service_roles
  name        = "${each.key}-role"
  description = each.value.description

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "services" {
  for_each = {
    for item in flatten([
      for role, config in var.service_roles : [
        for arn in config.policy_arns : {
          key  = "${role}-${basename(arn)}"
          role = role
          arn  = arn
        }
      ]
    ]) : item.key => item
  }

  role       = aws_iam_role.services[each.value.role].name
  policy_arn = each.value.arn
}
```

## Conclusion

Managing IAM users and roles with OpenTofu gives you an auditable, version-controlled record of every identity in your AWS environment. Follow least-privilege principles, use groups for user permissions, and prefer roles over users for machine identities.
