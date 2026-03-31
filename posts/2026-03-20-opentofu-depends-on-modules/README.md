# How to Use depends_on with Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn how to use depends_on with modules in OpenTofu to express explicit ordering dependencies when implicit dependency detection is insufficient.

## Introduction

OpenTofu automatically detects dependencies between resources when one resource's attribute references another. However, sometimes the dependency relationship exists but isn't captured through direct attribute references - for example, when a module depends on an IAM policy being attached before it can execute. In these cases, use `depends_on` on the module block.

## Syntax

```hcl
module "example" {
  source = "./modules/my-module"

  depends_on = [
    aws_iam_role_policy_attachment.my_attachment,
    module.prerequisites
  ]
}
```

## When to Use depends_on

Use `depends_on` on modules when:
1. A module needs a resource to be in a specific state that isn't expressed through attributes
2. IAM permissions must propagate before resources can be created
3. A prerequisite module must complete before this module starts
4. Indirect dependencies that OpenTofu can't auto-detect

## Practical Use Cases

### Waiting for IAM Policy to Propagate

```hcl
resource "aws_iam_role" "ecs_task" {
  name               = "ecs-task-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "s3_access" {
  role       = aws_iam_role.ecs_task.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

module "ecs_service" {
  source = "./modules/ecs-service"

  task_role_arn = aws_iam_role.ecs_task.arn
  image         = var.docker_image

  # Ensure the policy is attached before the ECS service tries to use it
  depends_on = [
    aws_iam_role_policy_attachment.s3_access
  ]
}
```

### Module Prerequisite Ordering

```hcl
module "networking" {
  source = "./modules/networking"

  cidr_block = "10.0.0.0/16"
}

module "security" {
  source = "./modules/security"

  vpc_id = module.networking.vpc_id
}

module "compute" {
  source = "./modules/compute"

  subnet_ids         = module.networking.private_subnet_ids
  security_group_ids = module.security.app_sg_ids

  # Depends on both networking and security being fully ready
  depends_on = [
    module.networking,
    module.security
  ]
}
```

### Database Before Application

```hcl
module "database" {
  source = "./modules/rds"

  identifier     = "app-db"
  instance_class = "db.t3.medium"
  subnet_ids     = module.vpc.private_subnet_ids
}

module "application" {
  source = "./modules/ecs"

  db_endpoint = module.database.endpoint
  image       = var.app_image

  # Application should start after the DB is fully ready
  depends_on = [module.database]
}
```

### Kubernetes Resources After CRDs

```hcl
module "cert_manager" {
  source = "./modules/cert-manager"
}

module "certificates" {
  source = "./modules/tls-certificates"

  # CRDs from cert-manager must exist before creating Certificate resources
  depends_on = [module.cert_manager]

  domain = var.domain
}
```

## Important: Use Sparingly

`depends_on` creates a dependency on the entire module, forcing full evaluation order. This can:
- Reduce plan parallelism (slowing down operations)
- Create unexpected re-evaluation chains

Prefer using output attributes as arguments when possible, as OpenTofu detects those automatically.

```hcl
# Preferred: use module output as input (auto-detects dependency)

module "app" {
  subnet_ids = module.vpc.private_subnet_ids  # Implicit dependency
}

# Use depends_on only when the dependency can't be expressed through attributes
module "app" {
  depends_on = [module.vpc]  # Explicit dependency
}
```

## Conclusion

The `depends_on` meta-argument for modules in OpenTofu provides a safety valve for expressing dependencies that can't be captured through attribute references. Use it sparingly - only when you have a genuine implicit dependency. Prefer attribute-based references which create more efficient, targeted dependencies.
