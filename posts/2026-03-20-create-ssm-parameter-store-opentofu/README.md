# How to Create AWS SSM Parameter Store Parameters with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, SSM, Parameter Store, Infrastructure as Code

Description: Learn how to create AWS Systems Manager Parameter Store parameters with OpenTofu for configuration management and secret storage in hierarchical namespaces.

SSM Parameter Store provides hierarchical configuration storage with optional encryption. It's lighter than Secrets Manager for non-rotating configuration and works seamlessly with EC2, Lambda, ECS, and EKS. Managing parameters in OpenTofu keeps your configuration hierarchy documented.

## Plain String Parameter

```hcl
resource "aws_ssm_parameter" "app_name" {
  name  = "/production/myapp/APP_NAME"
  type  = "String"
  value = "myapp"

  description = "Application name for environment variable injection"

  tags = {
    Environment = "production"
    Application = "myapp"
  }
}
```

## SecureString Parameters

```hcl
resource "aws_ssm_parameter" "db_password" {
  name   = "/production/myapp/DB_PASSWORD"
  type   = "SecureString"
  value  = var.db_password  # Pass via TF_VAR or secrets manager
  key_id = aws_kms_key.ssm.arn

  description = "Production database password"

  lifecycle {
    ignore_changes = [value]  # Don't overwrite manual rotations
  }
}
```

## StringList Parameter

```hcl
resource "aws_ssm_parameter" "allowed_ips" {
  name  = "/production/myapp/ALLOWED_IPS"
  type  = "StringList"
  value = join(",", var.allowed_ips)  # Comma-separated list

  description = "IP addresses allowed to access admin endpoints"
}
```

## Hierarchical Parameters with for_each

```hcl
locals {
  app_config = {
    DB_HOST          = aws_db_instance.main.address
    DB_PORT          = "5432"
    DB_NAME          = var.db_name
    REDIS_HOST       = aws_elasticache_replication_group.main.primary_endpoint_address
    REDIS_PORT       = "6379"
    S3_BUCKET        = aws_s3_bucket.data.id
    AWS_REGION       = var.region
    LOG_LEVEL        = "info"
    FEATURE_ENABLED  = "true"
  }
}

resource "aws_ssm_parameter" "app_config" {
  for_each = local.app_config

  name  = "/production/myapp/${each.key}"
  type  = "String"
  value = each.value

  tags = {
    Environment = "production"
    Application = "myapp"
  }
}
```

## SSM Parameter Versions (Pinned Config)

```hcl
resource "aws_ssm_parameter" "feature_flags" {
  name      = "/production/myapp/FEATURE_FLAGS"
  type      = "String"
  value     = jsonencode({
    new_checkout   = true
    dark_mode      = false
    ai_suggestions = false
  })
  data_type = "text"

  description = "Feature flag configuration"
}
```

## IAM Permission to Read Parameters

```hcl
resource "aws_iam_role_policy" "read_ssm_params" {
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath",
        ]
        Resource = "arn:aws:ssm:${var.region}:${data.aws_caller_identity.current.account_id}:parameter/production/myapp/*"
      },
      {
        Effect   = "Allow"
        Action   = "kms:Decrypt"
        Resource = aws_kms_key.ssm.arn
      }
    ]
  })
}
```

## Reading Parameters in ECS Task

```hcl
# ECS task definition references SSM parameters as environment variables

resource "aws_ecs_task_definition" "app" {
  family             = "myapp"
  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.app.arn
  network_mode       = "awsvpc"

  container_definitions = jsonencode([{
    name = "app"
    image = "${aws_ecr_repository.api.repository_url}:latest"

    secrets = [
      {
        name      = "DB_PASSWORD"
        valueFrom = aws_ssm_parameter.db_password.arn
      }
    ]

    environment = [
      {
        name  = "DB_HOST"
        value = aws_ssm_parameter.app_config["DB_HOST"].value
      }
    ]
  }])
}
```

## Conclusion

SSM Parameter Store in OpenTofu provides lightweight, hierarchical configuration management. Use String type for non-sensitive config, SecureString with a KMS key for sensitive values, and organize parameters in a `/environment/app/KEY` hierarchy for easy permission scoping. Use lifecycle ignore_changes on SecureString values to allow out-of-band rotation without OpenTofu reverting the value.
