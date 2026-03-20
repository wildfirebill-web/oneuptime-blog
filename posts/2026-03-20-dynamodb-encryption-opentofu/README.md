# How to Configure DynamoDB Encryption with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, DynamoDB, Encryption, KMS, Security, Infrastructure as Code

Description: Learn how to configure DynamoDB server-side encryption with customer-managed KMS keys using OpenTofu to control encryption key lifecycle and access policies.

## Introduction

DynamoDB encrypts all data at rest by default using AWS-owned keys. For compliance requirements or key management control, you can use AWS-managed keys (aws/dynamodb) or customer-managed KMS keys (CMK). CMKs provide key rotation control, detailed CloudTrail audit logging, and the ability to revoke access by disabling the key.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with DynamoDB and KMS permissions

## Step 1: Create Customer-Managed KMS Key for DynamoDB

```hcl
resource "aws_kms_key" "dynamodb" {
  description             = "KMS key for DynamoDB encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow DynamoDB to use the key"
        Effect = "Allow"
        Principal = {
          Service = "dynamodb.amazonaws.com"
        }
        Action = [
          "kms:DescribeKey",
          "kms:CreateGrant",
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
        Condition = {
          StringEquals = {
            "kms:CallerAccount"   = data.aws_caller_identity.current.account_id
            "kms:ViaService"      = "dynamodb.${var.region}.amazonaws.com"
          }
        }
      }
    ]
  })

  tags = {
    Name    = "${var.project_name}-dynamodb-key"
    Purpose = "DynamoDB"
  }
}

resource "aws_kms_alias" "dynamodb" {
  name          = "alias/${var.project_name}-dynamodb"
  target_key_id = aws_kms_key.dynamodb.key_id
}
```

## Step 2: Create DynamoDB Table with CMK Encryption

```hcl
resource "aws_dynamodb_table" "encrypted" {
  name         = "${var.project_name}-encrypted-table"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"

  attribute {
    name = "id"
    type = "S"
  }

  # Use customer-managed KMS key for encryption
  server_side_encryption {
    enabled     = true
    kms_key_arn = aws_kms_key.dynamodb.arn
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Name      = "${var.project_name}-encrypted-table"
    Encrypted = "CMK"
  }
}
```

## Step 3: Migrate Existing Table to CMK Encryption

```bash
# You can change encryption type for existing tables without downtime
aws dynamodb update-table \
  --table-name my-existing-table \
  --sse-specification "Enabled=true,SSEType=KMS,KMSMasterKeyId=alias/my-project-dynamodb"

# Monitor the update
aws dynamodb describe-table \
  --table-name my-existing-table \
  --query 'Table.{Status: TableStatus, SSEDescription: SSEDescription}'
```

## Step 4: Grant Application Access to Encrypted Table

```hcl
# Application IAM role needs both DynamoDB and KMS permissions
resource "aws_iam_role_policy" "app_dynamodb" {
  name = "dynamodb-encrypted-access"
  role = var.application_role_name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ]
        Resource = aws_dynamodb_table.encrypted.arn
      },
      {
        # Required for operations on CMK-encrypted tables
        Effect = "Allow"
        Action = [
          "kms:GenerateDataKey",
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = aws_kms_key.dynamodb.arn
      }
    ]
  })
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply

# Verify encryption configuration
aws dynamodb describe-table \
  --table-name my-project-encrypted-table \
  --query 'Table.SSEDescription'
```

## Conclusion

Customer-managed KMS keys for DynamoDB provide the highest level of control over encryption, including the ability to audit all key usage via CloudTrail and revoke access by disabling the key. The encryption is transparent to applications—no code changes required—but applications need both DynamoDB and KMS permissions when using CMKs. Enable key rotation annually to limit the blast radius of key compromise.
