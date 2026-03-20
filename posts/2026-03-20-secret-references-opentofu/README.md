# How to Use Secret References in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Secrets, Configuration, AWS, Infrastructure as Code

Description: Learn how to reference secrets from AWS Secrets Manager and SSM Parameter Store in OpenTofu configurations without storing sensitive values in state.

Referencing secrets properly in OpenTofu means the secret value flows from a secrets store to its destination without appearing in your code or being committed to version control. OpenTofu can wire the ARN/ID of a secret to a service without needing the value itself.

## Referencing Secrets by ARN in ECS

```hcl
# The secret exists — we reference it by ARN
data "aws_secretsmanager_secret" "db" {
  name = "production/myapp/database"
}

resource "aws_ecs_task_definition" "app" {
  family             = "myapp"
  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.app.arn
  network_mode       = "awsvpc"

  container_definitions = jsonencode([{
    name  = "app"
    image = "${aws_ecr_repository.api.repository_url}:latest"

    # Inject secrets as environment variables at task start
    # The value never appears in the task definition — only the ARN
    secrets = [
      {
        name      = "DB_PASSWORD"
        # Reference specific key from JSON secret
        valueFrom = "${data.aws_secretsmanager_secret.db.arn}:password::"
      },
      {
        name      = "DB_HOST"
        valueFrom = "${data.aws_secretsmanager_secret.db.arn}:host::"
      },
    ]
  }])
}
```

## SSM Parameter References in ECS

```hcl
data "aws_ssm_parameter" "api_key" {
  name = "/production/myapp/STRIPE_API_KEY"
}

resource "aws_ecs_task_definition" "app" {
  family             = "myapp"
  execution_role_arn = aws_iam_role.ecs_execution.arn
  network_mode       = "awsvpc"

  container_definitions = jsonencode([{
    name  = "app"
    image = "${aws_ecr_repository.api.repository_url}:latest"

    secrets = [
      {
        name      = "STRIPE_API_KEY"
        valueFrom = data.aws_ssm_parameter.api_key.arn
      }
    ]
  }])
}
```

## Secret References in Lambda

```hcl
resource "aws_lambda_function" "app" {
  function_name = "myapp"
  runtime       = "python3.12"
  handler       = "index.handler"
  filename      = "app.zip"
  role          = aws_iam_role.lambda.arn

  environment {
    variables = {
      # Pass the secret ARN — Lambda code reads the value at runtime
      SECRET_ARN    = aws_secretsmanager_secret.db_credentials.arn
      DB_SECRET_NAME = aws_secretsmanager_secret.db_credentials.name
    }
  }
}
```

## Secret References in Kubernetes (External Secrets Operator)

```hcl
# Create ExternalSecret CRD that maps AWS Secrets Manager to Kubernetes Secrets
resource "kubectl_manifest" "external_secret" {
  yaml_body = <<-YAML
    apiVersion: external-secrets.io/v1beta1
    kind: ExternalSecret
    metadata:
      name: db-credentials
      namespace: production
    spec:
      refreshInterval: 1h
      secretStoreRef:
        name: aws-secretsmanager
        kind: ClusterSecretStore
      target:
        name: db-credentials
      data:
        - secretKey: password
          remoteRef:
            key: ${aws_secretsmanager_secret.db_credentials.name}
            property: password
        - secretKey: host
          remoteRef:
            key: ${aws_secretsmanager_secret.db_credentials.name}
            property: host
  YAML
}
```

## Sensitive Variable References

```hcl
# Mark outputs as sensitive to prevent accidental logging
output "db_connection_string" {
  value     = "postgresql://${var.db_username}@${aws_db_instance.main.address}:5432/${var.db_name}"
  sensitive = true  # Will show [sensitive] in plan/apply output
}

# Reference a data source without exposing the value in state
data "aws_secretsmanager_secret_version" "db" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
}

locals {
  # jsondecode is available but the value is still sensitive
  db_config = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)
}
```

## Conclusion

Secret references in OpenTofu keep sensitive values out of configuration files. Pass secret ARNs to ECS, Lambda, and Kubernetes rather than injecting values directly — services retrieve them at runtime. Use sensitive = true on outputs that might contain derived secret values to prevent accidental exposure in logs and CI/CD output.
