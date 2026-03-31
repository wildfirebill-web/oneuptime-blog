# How to Use depends_on for Explicit Dependencies in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Resource, depends_on, Dependencies, Infrastructure as Code, DevOps

Description: A guide to using depends_on to create explicit dependency relationships between OpenTofu resources and modules.

## Introduction

OpenTofu automatically determines dependencies between resources based on reference expressions. However, some dependencies are implicit - they exist due to side effects not captured by references. The `depends_on` meta-argument lets you declare these hidden dependencies explicitly.

## When depends_on Is Needed

```hcl
# Automatic dependency (no depends_on needed):

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id  # Reference creates automatic dependency
}

# Explicit dependency (depends_on needed):
resource "null_resource" "database_setup" {
  provisioner "local-exec" {
    command = "python setup_db.py"
  }
}

resource "aws_ecs_service" "app" {
  name    = "app"
  cluster = aws_ecs_cluster.main.id

  # App needs database setup to be complete
  # but doesn't reference null_resource.database_setup directly
  depends_on = [null_resource.database_setup]
}
```

## Basic depends_on Usage

```hcl
# Resource dependency
resource "aws_iam_role_policy" "app_policy" {
  name   = "app-policy"
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.app.json
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  iam_instance_profile = aws_iam_instance_profile.app.name

  # Ensure IAM policy is attached before creating instance
  # (even though instance doesn't directly reference the policy)
  depends_on = [aws_iam_role_policy.app_policy]
}
```

## Module depends_on

```hcl
module "database" {
  source = "./modules/rds"
  # ...
}

module "migration" {
  source = "./modules/db-migration"

  db_endpoint = module.database.endpoint
  depends_on  = [module.database]  # Wait for entire module to complete
}

module "application" {
  source = "./modules/ecs-service"

  db_endpoint = module.database.endpoint

  # App needs migration to complete first
  depends_on = [module.migration]
}
```

## Kubernetes Example

```hcl
resource "kubernetes_namespace" "app" {
  metadata {
    name = "app"
  }
}

resource "kubernetes_config_map" "app_config" {
  metadata {
    name      = "app-config"
    namespace = kubernetes_namespace.app.metadata[0].name
  }

  data = {
    DATABASE_URL = var.database_url
  }
}

resource "kubernetes_deployment" "app" {
  metadata {
    name      = "app"
    namespace = kubernetes_namespace.app.metadata[0].name
  }

  spec {
    # ... deployment spec ...
  }

  # Ensure ConfigMap exists before deploying
  depends_on = [kubernetes_config_map.app_config]
}
```

## depends_on with Data Sources

```hcl
resource "aws_s3_bucket_policy" "logs" {
  bucket = aws_s3_bucket.logs.id
  policy = data.aws_iam_policy_document.logs.json
}

# Data source that reads the policy we just applied
data "aws_s3_bucket" "verified_logs" {
  bucket = aws_s3_bucket.logs.id

  # Read after policy is applied to verify configuration
  depends_on = [aws_s3_bucket_policy.logs]
}
```

## Multiple Dependencies

```hcl
resource "aws_ecs_service" "app" {
  name            = "app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2

  # App needs multiple things to be ready:
  depends_on = [
    aws_iam_role_policy.ecs_execution,      # IAM role must have permissions
    aws_lb_listener.https,                   # Load balancer must be listening
    aws_security_group_rule.app_inbound,     # Security rules must be applied
    null_resource.database_migration,        # DB migration must complete
  ]
}
```

## When NOT to Use depends_on

```hcl
# WRONG: Using depends_on when a reference creates the dependency
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"

  # UNNECESSARY - the vpc_id reference already creates the dependency
  # depends_on = [aws_vpc.main]  # Remove this
}

# CORRECT: Direct reference is sufficient
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

## Conclusion

`depends_on` is a powerful tool for expressing dependencies that OpenTofu cannot automatically detect from resource references. It's particularly important for resources with side effects (like `null_resource` provisioners), IAM policy attachments, and any scenario where one resource's completion affects another's functionality without a direct reference. Use it sparingly - unnecessary `depends_on` slows down deployments by preventing parallelism that would otherwise be safe.
