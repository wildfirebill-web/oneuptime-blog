# How to Override Module Calls in OpenTofu Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Override Modules, Mock, Infrastructure as Code

Description: Learn how to use `override_module` in OpenTofu tests to replace child module calls with controlled output values, enabling isolated unit testing of parent modules.

## Introduction

When testing a parent module that calls child modules, you often want to isolate the parent's logic without executing the child modules. OpenTofu's `override_module` block lets you replace a child module call with a set of controlled output values, enabling true unit testing of module composition.

## Basic `override_module` Syntax

```hcl
# tests/parent_module.tftest.hcl

mock_provider "aws" {}

run "parent_uses_networking_outputs" {
  # Replace the entire networking module call with fixed outputs
  override_module {
    target = module.networking

    outputs = {
      vpc_id          = "vpc-override-12345"
      private_subnets = ["subnet-aaa", "subnet-bbb"]
      public_subnets  = ["subnet-ccc", "subnet-ddd"]
    }
  }

  assert {
    condition     = aws_eks_cluster.this.vpc_config[0].vpc_id == "vpc-override-12345"
    error_message = "EKS cluster should use the VPC ID from the networking module"
  }
}
```

## File-Level Module Overrides

Apply an override to all `run` blocks by placing it at the file level:

```hcl
# Override the shared services module for all tests in this file
override_module {
  target = module.shared_services

  outputs = {
    kms_key_arn       = "arn:aws:kms:us-east-1:123456789012:key/override-key"
    log_bucket_name   = "override-log-bucket"
    monitoring_topic  = "arn:aws:sns:us-east-1:123456789012:override-alerts"
  }
}

run "application_uses_shared_kms_key" {
  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.app.rule[0].apply_server_side_encryption_by_default[0].kms_master_key_id == "arn:aws:kms:us-east-1:123456789012:key/override-key"
    error_message = "Application should use the KMS key from the shared services module"
  }
}
```

## Overriding Nested Module Calls

For deeply nested modules, use the full dotted path:

```hcl
run "nested_module_override" {
  # Override a module nested inside another module
  override_module {
    target = module.application.module.database

    outputs = {
      endpoint = "override-db.cluster-xyz.us-east-1.rds.amazonaws.com"
      port     = 5432
      name     = "appdb"
    }
  }

  assert {
    condition     = aws_ecs_task_definition.app.container_definitions != ""
    error_message = "Task definition should be created"
  }
}
```

## Practical Example: Three-Tier Architecture

Consider a root module that composes networking, compute, and database child modules:

```hcl
# Root module structure:
# module "networking" { ... }
# module "compute" { depends_on = [module.networking] ... }
# module "database" { depends_on = [module.networking, module.compute] ... }
```

Testing only the compute module's logic:

```hcl
# tests/compute.tftest.hcl

mock_provider "aws" {}

run "compute_uses_correct_subnet" {
  # Stub out networking — we're only testing compute logic
  override_module {
    target = module.networking

    outputs = {
      vpc_id          = "vpc-stub-001"
      private_subnets = ["subnet-private-001", "subnet-private-002"]
      alb_sg_id       = "sg-alb-stub-001"
    }
  }

  # Stub out the database — not relevant to this test
  override_module {
    target = module.database

    outputs = {
      endpoint = "stub-db.local"
      port     = 5432
    }
  }

  assert {
    condition     = module.compute.asg_subnet_ids == toset(["subnet-private-001", "subnet-private-002"])
    error_message = "ASG should launch into the private subnets from the networking module"
  }
}
```

## When to Use `override_module`

- Testing a parent module in isolation from expensive or slow child modules.
- Breaking circular test dependencies where child modules require credentials.
- Verifying that your parent module correctly passes outputs between child modules.
- Reducing test execution time by mocking infrastructure-heavy child modules.

## Conclusion

`override_module` completes the OpenTofu testing toolkit for module composition. Combined with `mock_provider`, `override_resource`, and `override_data`, it allows you to test any layer of your infrastructure stack in complete isolation—fast, cheap, and without cloud credentials.
