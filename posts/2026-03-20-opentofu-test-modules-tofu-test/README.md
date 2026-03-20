# How to Test Modules with tofu test in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Modules, Testing

Description: Learn how to use the tofu test command in OpenTofu to write automated tests for your infrastructure modules with assertions and mock providers.

## Introduction

OpenTofu includes a built-in testing framework via the `tofu test` command. Tests are written in `.tftest.hcl` files and can create real infrastructure, run assertions against it, and clean up automatically - or use mock providers to test logic without creating any resources.

## Test File Structure

Create a `tests/` directory inside your module:

```text
terraform-aws-vpc/
├── main.tf
├── variables.tf
├── outputs.tf
└── tests/
    ├── basic.tftest.hcl
    └── complete.tftest.hcl
```

## Basic Test File

```hcl
# tests/basic.tftest.hcl

# Optional: override variables for the test

variables {
  name       = "test-vpc"
  cidr_block = "10.0.0.0/16"
}

# A run block creates infrastructure and runs assertions
run "creates_vpc" {
  command = plan  # Use "plan" for cheap tests; "apply" to test real resources

  assert {
    condition     = aws_vpc.this.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block does not match the expected value"
  }

  assert {
    condition     = aws_vpc.this.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}
```

## Running Tests

```bash
# Run all tests in the current module
tofu test

# Run a specific test file
tofu test -filter=tests/basic.tftest.hcl

# Verbose output
tofu test -verbose
```

## Testing with Real Infrastructure (apply)

```hcl
# tests/integration.tftest.hcl
run "creates_real_vpc" {
  command = apply  # Actually creates infrastructure

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID must be non-empty after apply"
  }
}

# OpenTofu automatically destroys test infrastructure after the test run
```

## Using Mock Providers

Mock providers avoid creating real resources and make tests fast:

```hcl
# tests/unit.tftest.hcl

mock_provider "aws" {
  mock_resource "aws_vpc" {
    defaults = {
      id         = "vpc-mock-12345"
      arn        = "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-mock-12345"
    }
  }
}

variables {
  name       = "test-vpc"
  cidr_block = "10.0.0.0/16"
}

run "validates_outputs" {
  assert {
    condition     = output.vpc_id == "vpc-mock-12345"
    error_message = "VPC output ID mismatch"
  }
}
```

## Testing Variable Validation

Test that your validation rules work correctly:

```hcl
# tests/validation.tftest.hcl

run "rejects_invalid_cidr" {
  command = plan

  variables {
    name       = "test"
    cidr_block = "invalid-cidr"  # Should fail validation
  }

  expect_failures = [var.cidr_block]  # Expect this variable to fail validation
}
```

## Multiple Run Blocks

Chain multiple test scenarios in one file:

```hcl
# tests/scenarios.tftest.hcl

run "minimal_config" {
  variables {
    name = "minimal"
  }

  assert {
    condition     = aws_vpc.this.enable_dns_support == true
    error_message = "DNS support should default to true"
  }
}

run "custom_tags" {
  variables {
    name = "tagged"
    tags = { Team = "platform" }
  }

  assert {
    condition     = aws_vpc.this.tags["Team"] == "platform"
    error_message = "Custom tags not applied correctly"
  }
}
```

## Conclusion

`tofu test` brings first-class testing to OpenTofu modules. Use `plan` tests for fast, cheap validation of resource configurations. Use `apply` tests for integration testing against real cloud APIs. Use mock providers for unit testing logic without cloud credentials. Combine all three in a CI/CD pipeline to catch regressions before modules are published or deployed.
