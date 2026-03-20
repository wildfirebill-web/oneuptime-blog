# How to Configure AWS Systems Manager Parameter Store with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, SSM, Parameter Store, Secrets Management, Configuration, Infrastructure as Code

Description: Learn how to store and retrieve application configuration and secrets using AWS Systems Manager Parameter Store with OpenTofu, including SecureString parameters encrypted with KMS.

## Introduction

AWS Systems Manager Parameter Store provides secure, hierarchical storage for configuration data and secrets. It supports plain text (String, StringList) and encrypted (SecureString) parameter types, integrates natively with EC2, Lambda, ECS, and EKS for automatic secret injection, and offers free standard parameters or paid advanced parameters for large values and higher throughput.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with SSM Parameter Store permissions

## Step 1: Create Plain Text Configuration Parameters

```hcl
# Application configuration parameters (non-sensitive)
resource "aws_ssm_parameter" "app_environment" {
  name  = "/${var.project_name}/${var.environment}/config/APP_ENV"
  type  = "String"
  value = var.environment

  tags = {
    Name        = "${var.project_name}-app-env"
    Environment = var.environment
  }
}

resource "aws_ssm_parameter" "database_url" {
  name  = "/${var.project_name}/${var.environment}/config/DATABASE_HOST"
  type  = "String"
  value = var.database_host  # Non-sensitive hostname, not credentials
}

# StringList for multi-value parameters
resource "aws_ssm_parameter" "allowed_origins" {
  name  = "/${var.project_name}/${var.environment}/config/ALLOWED_ORIGINS"
  type  = "StringList"
  value = join(",", var.allowed_origins)  # e.g., "https://app.example.com,https://api.example.com"
}
```

## Step 2: Create Encrypted SecureString Parameters

```hcl
resource "aws_ssm_parameter" "database_password" {
  name   = "/${var.project_name}/${var.environment}/secrets/DATABASE_PASSWORD"
  type   = "SecureString"
  value  = var.database_password  # Mark as sensitive in Terraform
  key_id = var.kms_key_arn        # Use customer-managed KMS key

  tags = {
    Name        = "${var.project_name}-db-password"
    Environment = var.environment
    Sensitive   = "true"
  }

  lifecycle {
    ignore_changes = [value]  # Don't overwrite if value changed outside Terraform
  }
}

resource "aws_ssm_parameter" "api_key" {
  name   = "/${var.project_name}/${var.environment}/secrets/EXTERNAL_API_KEY"
  type   = "SecureString"
  value  = var.external_api_key
  key_id = var.kms_key_arn

  tags = {
    Name = "${var.project_name}-api-key"
  }

  lifecycle {
    ignore_changes = [value]
  }
}
```

## Step 3: IAM Policy for Parameter Access

```hcl
# Policy for application to read parameters
resource "aws_iam_policy" "ssm_read" {
  name = "${var.project_name}-ssm-read"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath"
        ]
        Resource = [
          "arn:aws:ssm:${var.region}:${data.aws_caller_identity.current.account_id}:parameter/${var.project_name}/${var.environment}/*"
        ]
      },
      {
        # Required for SecureString parameters with customer-managed KMS key
        Effect   = "Allow"
        Action   = ["kms:Decrypt"]
        Resource = var.kms_key_arn
      }
    ]
  })
}
```

## Step 4: Read Parameters in Application Code

```python
import boto3
import os

ssm = boto3.client('ssm', region_name='us-east-1')

def get_config():
    """Load all parameters for this service from Parameter Store."""
    prefix = f"/{os.environ['PROJECT_NAME']}/{os.environ['ENVIRONMENT']}"

    # Get all parameters under the prefix
    response = ssm.get_parameters_by_path(
        Path=f"{prefix}/config/",
        Recursive=True,
        WithDecryption=False
    )

    # Get secrets
    secret_response = ssm.get_parameters_by_path(
        Path=f"{prefix}/secrets/",
        Recursive=True,
        WithDecryption=True  # Required for SecureString
    )

    config = {}
    for param in response['Parameters'] + secret_response['Parameters']:
        # Extract the parameter name without the path prefix
        key = param['Name'].split('/')[-1]
        config[key] = param['Value']

    return config
```

## Step 5: Use Parameters in Lambda

```hcl
# Lambda environment variable referencing SSM parameter
resource "aws_lambda_function" "app" {
  # ...
  environment {
    variables = {
      # Reference SSM parameter ARN for Lambda to load at startup
      DB_PASSWORD_SSM_PATH = aws_ssm_parameter.database_password.name
      CONFIG_PREFIX        = "/${var.project_name}/${var.environment}"
    }
  }
}
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply

# Read a parameter value
aws ssm get-parameter \
  --name "/my-project/prod/secrets/DATABASE_PASSWORD" \
  --with-decryption \
  --query 'Parameter.Value' --output text
```

## Conclusion

Use a hierarchical naming convention like `/{project}/{environment}/{type}/{name}` to organize parameters and enable path-based IAM policies that grant least-privilege access. Use `lifecycle { ignore_changes = [value] }` for secrets managed outside Terraform to prevent accidental overwrites during deployments. For Lambda and ECS, load parameters at startup using `GetParametersByPath` to reduce SSM API calls and improve performance.
