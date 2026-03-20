# How to Manage Database Schema Migrations with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Database, Schema Migrations, RDS, PostgreSQL, Infrastructure as Code

Description: Learn how to integrate database schema migrations with OpenTofu using null resources, local-exec, and migration tools like Flyway and Liquibase.

## Introduction

OpenTofu manages infrastructure, but database schema changes need to be coordinated with it. This guide shows patterns for running schema migrations as part of your OpenTofu workflow using null resources, Lambda, and dedicated migration tools.

## Using null_resource with local-exec

Run Flyway or Liquibase migrations after the database is provisioned.

```hcl
resource "aws_db_instance" "postgres" {
  identifier        = "${var.app_name}-db-${var.environment}"
  engine            = "postgres"
  engine_version    = "16.2"
  instance_class    = "db.t3.medium"
  username          = var.db_username
  password          = var.db_password
  allocated_storage = 20
  db_name           = var.db_name

  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  skip_final_snapshot = var.environment != "prod"
}

# Run Flyway migrations after DB is ready
resource "null_resource" "db_migrations" {
  depends_on = [aws_db_instance.postgres]

  # Re-run migrations whenever migration files change
  triggers = {
    migrations_hash = sha256(join("", [for f in fileset("${path.module}/migrations", "*.sql") : filesha256("${path.module}/migrations/${f}")]))
    db_endpoint     = aws_db_instance.postgres.endpoint
  }

  provisioner "local-exec" {
    command = <<-EOT
      flyway \
        -url="jdbc:postgresql://${aws_db_instance.postgres.endpoint}/${var.db_name}" \
        -user="${var.db_username}" \
        -password="${var.db_password}" \
        -locations="filesystem:${path.module}/migrations" \
        migrate
    EOT
  }
}
```

## Migration Files Structure

```
migrations/
├── V1__initial_schema.sql
├── V2__add_users_table.sql
├── V3__add_orders_index.sql
└── V4__add_audit_columns.sql
```

## Example Migration File

```sql
-- migrations/V2__add_users_table.sql
-- Create the users table with audit columns

CREATE TABLE IF NOT EXISTS users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) NOT NULL UNIQUE,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- Trigger to update updated_at automatically
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

## Lambda-Based Migration Runner

For environments where local-exec is not suitable (remote backends, CI/CD).

```hcl
resource "aws_lambda_function" "db_migrator" {
  function_name = "${var.app_name}-db-migrator"
  runtime       = "python3.12"
  handler       = "migrator.handler"
  role          = aws_iam_role.migrator.arn
  filename      = "migrator.zip"
  timeout       = 300
  memory_size   = 512

  vpc_config {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.lambda.id]
  }

  environment {
    variables = {
      DB_HOST     = aws_db_instance.postgres.address
      DB_PORT     = aws_db_instance.postgres.port
      DB_NAME     = var.db_name
      SECRET_ARN  = aws_secretsmanager_secret.db.arn
    }
  }
}

# Invoke the Lambda to run migrations after DB creation
resource "aws_lambda_invocation" "run_migrations" {
  function_name = aws_lambda_function.db_migrator.function_name

  input = jsonencode({
    action = "migrate"
    target = "latest"
  })

  depends_on = [aws_db_instance.postgres, aws_lambda_function.db_migrator]

  triggers = {
    migrations_version = var.migrations_version
  }
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

Database schema migrations should be coordinated with infrastructure changes. OpenTofu's null_resource with local-exec, triggers based on migration file hashes, and Lambda-based runners provide flexible patterns for running migrations as part of your infrastructure deployment pipeline.
