# How to Override Module Calls in OpenTofu Tests - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Module, Override, Infrastructure as Code

Description: Learn how to use override_module in OpenTofu tests to replace module outputs with controlled values, enabling testing of module composition without executing sub-modules.

## Introduction

When testing OpenTofu configurations that consume modules, you often want to test the calling configuration without executing the full sub-module. The `override_module` directive in OpenTofu tests lets you replace a module call with predefined output values, enabling isolated testing of module composition logic.

## Why Override Module Calls

Module overrides are useful when:
- A sub-module provisions expensive or slow-to-create resources
- You want to test how the parent configuration handles module outputs
- The sub-module has its own separate tests
- You're writing unit tests for a root configuration that uses modules

## Basic Module Override

Given a configuration that uses a VPC module:

```hcl
# main.tf

module "vpc" {
  source = "./modules/vpc"

  cidr_block   = var.vpc_cidr
  environment  = var.environment
  azs          = var.availability_zones
}

module "eks" {
  source = "./modules/eks"

  cluster_name    = "app-${var.environment}"
  vpc_id          = module.vpc.vpc_id
  private_subnets = module.vpc.private_subnet_ids
}

resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = module.vpc.vpc_id
}
```

Override the VPC module in tests:

```hcl
# tests/composition.tftest.hcl

run "eks_uses_vpc_module_outputs" {
  command = plan

  override_module {
    target = module.vpc
    outputs = {
      vpc_id            = "vpc-mock12345"
      private_subnet_ids = ["subnet-private1", "subnet-private2", "subnet-private3"]
      public_subnet_ids  = ["subnet-public1", "subnet-public2"]
      cidr_block         = "10.0.0.0/16"
    }
  }

  # Also override eks to avoid running it
  override_module {
    target = module.eks
    outputs = {
      cluster_name     = "app-testing"
      cluster_endpoint = "https://mock-eks-endpoint.example.com"
      cluster_arn      = "arn:aws:eks:us-east-1:123456789:cluster/app-testing"
    }
  }

  variables {
    vpc_cidr          = "10.0.0.0/16"
    environment       = "testing"
    availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  }

  assert {
    condition     = aws_security_group.app.vpc_id == "vpc-mock12345"
    error_message = "Security group should be in the VPC from the module"
  }
}
```

## Testing Module Output Consumption

Test that your configuration correctly uses module outputs:

```hcl
# main.tf

module "database" {
  source = "./modules/rds"

  engine         = "postgres"
  instance_class = local.db_size[var.environment]
  db_name        = "appdb"
}

locals {
  db_size = {
    dev     = "db.t3.micro"
    staging = "db.t3.small"
    prod    = "db.r5.large"
  }
}

resource "aws_ssm_parameter" "db_endpoint" {
  name  = "/app/${var.environment}/db-endpoint"
  type  = "String"
  value = module.database.endpoint
}
```

```hcl
# tests/db_config.tftest.hcl

run "db_endpoint_stored_in_ssm" {
  command = plan

  override_module {
    target = module.database
    outputs = {
      endpoint = "mock-rds.cluster.us-east-1.rds.amazonaws.com"
      port     = 5432
      db_name  = "appdb"
    }
  }

  variables {
    environment = "staging"
  }

  assert {
    condition     = aws_ssm_parameter.db_endpoint.value == "mock-rds.cluster.us-east-1.rds.amazonaws.com"
    error_message = "SSM parameter should store the database endpoint"
  }

  assert {
    condition     = aws_ssm_parameter.db_endpoint.name == "/app/staging/db-endpoint"
    error_message = "SSM parameter path should include environment"
  }
}
```

## Nested Module Overrides

Override modules at multiple levels:

```hcl
# Root uses module "platform"

# "platform" uses module "networking"

run "nested_module_override" {
  command = plan

  override_module {
    target = module.platform
    outputs = {
      vpc_id          = "vpc-platform"
      cluster_endpoint = "https://cluster.example.com"
    }
  }

  # Override nested module within platform
  # Note: Overrides are for the root configuration's direct module calls
}
```

## Comparing with Full Module Execution

Use separate test files for unit (override) vs. integration (full):

```hcl
# tests/unit.tftest.hcl - fast, offline
run "unit_test" {
  override_module {
    target = module.vpc
    outputs = { vpc_id = "vpc-unit" }
  }
  # ... assertions
}
```

```hcl
# tests/integration.tftest.hcl - slow, requires cloud access
run "integration_test" {
  # No overrides - uses real module execution
  variables {
    environment = "test"
  }
  # ... assertions on real resources
}
```

Run selectively:

```bash
# Unit tests only
tofu test -filter=tests/unit.tftest.hcl

# Integration tests
tofu test -filter=tests/integration.tftest.hcl
```

## Best Practices

- Override modules to test configuration logic; use real modules for integration tests.
- Set realistic output values that match the types your module actually returns.
- Name test runs descriptively: `override_vpc_outputs_flow_to_eks`.
- Test both the "happy path" and edge cases by varying override values across runs.
- Keep override values consistent with what the real module would return.

## Conclusion

The `override_module` directive enables targeted unit testing of module composition in OpenTofu. By controlling module outputs, you can test how your root configuration responds to different module results without executing expensive sub-module provisions, significantly speeding up your test suite.
