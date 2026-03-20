# How to Build a Secrets Management Infrastructure with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Secrets Management, AWS Secrets Manager, KMS, Security, Infrastructure as Code

Description: Learn how to build a comprehensive secrets management infrastructure on AWS with OpenTofu using Secrets Manager, KMS, automatic rotation, and cross-account access.

## Introduction

Secrets management infrastructure includes the storage layer (Secrets Manager), encryption layer (KMS), access control (IAM), rotation automation (Lambda), and the patterns for applications to retrieve secrets securely. This guide builds a production-grade secrets management setup.

## KMS Keys for Secret Encryption

```hcl
resource "aws_kms_key" "secrets" {
  description             = "KMS key for Secrets Manager"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnableRootAccess"
        Effect = "Allow"
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        Sid    = "AllowSecretsManagerUse"
        Effect = "Allow"
        Principal = { Service = "secretsmanager.amazonaws.com" }
        Action = ["kms:Encrypt", "kms:Decrypt", "kms:ReEncrypt*", "kms:GenerateDataKey*"]
        Resource = "*"
      }
    ]
  })
}

resource "aws_kms_alias" "secrets" {
  name          = "alias/myapp-secrets-${var.environment}"
  target_key_id = aws_kms_key.secrets.key_id
}
```

## Secrets Manager Secrets

```hcl
# Database credentials

resource "aws_secretsmanager_secret" "db" {
  name        = "myapp/${var.environment}/database"
  description = "Database connection credentials"
  kms_key_id  = aws_kms_key.secrets.arn

  recovery_window_in_days = var.environment == "prod" ? 30 : 7
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db.id

  secret_string = jsonencode({
    username = "myapp"
    password = var.db_initial_password  # rotated automatically after setup
    host     = aws_db_instance.main.address
    port     = 5432
    dbname   = "myapp"
  })

  lifecycle {
    ignore_changes = [secret_string]  # managed by rotation after initial setup
  }
}
```

## Automatic Secret Rotation

```hcl
resource "aws_secretsmanager_secret_rotation" "db" {
  secret_id           = aws_secretsmanager_secret.db.id
  rotation_lambda_arn = aws_lambda_function.db_rotation.arn

  rotation_rules {
    automatically_after_days = 30
  }
}

# Use AWS-provided rotation Lambda for RDS
resource "aws_lambda_function" "db_rotation" {
  function_name = "myapp-${var.environment}-db-rotation"
  description   = "Rotates RDS credentials in Secrets Manager"

  # AWS provides rotation lambdas for common databases
  # Deploy from the Serverless Application Repository
  s3_bucket         = "awsserverlessrepo-changesets-${data.aws_caller_identity.current.account_id}"
  s3_key            = "SecretsManagerRDSPostgreSQLRotationSingleUser"
  handler           = "lambda_function.lambda_handler"
  runtime           = "python3.11"
  role              = aws_iam_role.rotation_lambda.arn
  timeout           = 30
  vpc_config {
    subnet_ids         = module.vpc.private_subnets
    security_group_ids = [aws_security_group.rotation_lambda.id]
  }

  environment {
    variables = {
      SECRETS_MANAGER_ENDPOINT = "https://secretsmanager.${var.region}.amazonaws.com"
    }
  }
}
```

## Application Access to Secrets

```hcl
# IAM policy for application to read specific secrets
resource "aws_iam_policy" "app_secrets" {
  name = "myapp-${var.environment}-secrets-read"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [
          aws_secretsmanager_secret.db.arn,
          aws_secretsmanager_secret.api_keys.arn,
        ]
      },
      {
        Effect   = "Allow"
        Action   = ["kms:Decrypt"]
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

## Cross-Account Secret Access

```hcl
# Allow a workload in another account to access secrets
resource "aws_secretsmanager_secret_policy" "db" {
  secret_arn = aws_secretsmanager_secret.db.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.workload_account_id}:role/AppRole" }
      Action    = ["secretsmanager:GetSecretValue"]
      Resource  = "*"
    }]
  })
}
```

## Summary

A production secrets management infrastructure uses dedicated KMS keys for secret encryption with automatic key rotation, Secrets Manager for secret storage with recovery windows, automatic rotation for database passwords via rotation Lambdas, and least-privilege IAM policies for application access. Use JSON-structured secrets for database connections so a single secret contains all connection parameters. The `ignore_changes = [secret_string]` lifecycle rule on the initial secret version prevents OpenTofu from reverting rotated passwords.
