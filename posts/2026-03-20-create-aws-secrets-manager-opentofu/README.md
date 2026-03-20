# How to Create AWS Secrets Manager Secrets with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Secrets Manager, Security, Infrastructure as Code

Description: Learn how to create and manage AWS Secrets Manager secrets with OpenTofu for secure storage of database credentials, API keys, and application secrets.

AWS Secrets Manager stores and rotates secrets like database credentials and API keys. Managing the secret structure and access policies in OpenTofu ensures consistent configuration, while the actual secret values remain outside of version control.

## Creating a Secret

```hcl
resource "aws_secretsmanager_secret" "db_credentials" {
  name        = "production/myapp/database"
  description = "Production database credentials for myapp"
  kms_key_id  = aws_kms_key.secrets.arn

  # Recovery window: 0 to disable (immediate delete), or 7-30 days
  recovery_window_in_days = 7

  tags = {
    Environment = "production"
    Team        = "backend"
    Application = "myapp"
  }
}

# Set the secret value (reference from existing secret or variable)
# Note: store the actual value via console/CLI, or use sensitive variable
resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id

  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password
    host     = aws_db_instance.main.address
    port     = 5432
    dbname   = var.db_name
  })
}
```

## Secret Resource Policy

```hcl
resource "aws_secretsmanager_secret_policy" "db_credentials" {
  secret_arn = aws_secretsmanager_secret.db_credentials.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowAppAccess"
        Effect = "Allow"
        Principal = {
          AWS = [
            aws_iam_role.app.arn,
            aws_iam_role.worker.arn,
          ]
        }
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret",
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyNonVPC"
        Effect = "Deny"
        Principal = { AWS = "*" }
        Action   = "secretsmanager:GetSecretValue"
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:SourceVpc" = var.vpc_id
          }
        }
      }
    ]
  })
}
```

## Automatic Rotation

```hcl
resource "aws_secretsmanager_secret_rotation" "db_credentials" {
  secret_id           = aws_secretsmanager_secret.db_credentials.id
  rotation_lambda_arn = aws_lambda_function.secret_rotator.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

## IAM Permission to Read Secret

```hcl
resource "aws_iam_role_policy" "read_db_secret" {
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret",
        ]
        Resource = aws_secretsmanager_secret.db_credentials.arn
      },
      {
        Effect   = "Allow"
        Action   = "kms:Decrypt"
        Resource = aws_kms_key.secrets.arn
        Condition = {
          StringEquals = {
            "kms:ViaService" = "secretsmanager.${var.region}.amazonaws.com"
          }
        }
      }
    ]
  })
}
```

## Multiple Secrets with for_each

```hcl
locals {
  secrets = {
    "production/myapp/database"    = "Production database credentials"
    "production/myapp/redis"       = "Production Redis auth token"
    "production/myapp/stripe"      = "Stripe API keys"
    "production/myapp/sendgrid"    = "SendGrid API key"
  }
}

resource "aws_secretsmanager_secret" "app_secrets" {
  for_each    = local.secrets

  name        = each.key
  description = each.value
  kms_key_id  = aws_kms_key.secrets.arn

  recovery_window_in_days = 7

  tags = {
    Environment = "production"
  }
}

output "secret_arns" {
  value = {
    for k, v in aws_secretsmanager_secret.app_secrets : k => v.arn
  }
}
```

## Conclusion

AWS Secrets Manager in OpenTofu provides secure secret storage with access policies, encryption, and rotation. Define the secret structure and access policies in OpenTofu, but provide actual secret values through secure channels (environment variables, CI/CD vault, or the AWS console). Grant read access only from within your VPC and only to the specific roles that need each secret.
