# Passing Variables to OpenTofu Tests

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Variables, Infrastructure as Code, CI/CD

Description: Learn how to pass input variables to OpenTofu test files to make tests configurable, reusable, and environment-aware.

## Why Pass Variables to Tests?

Hardcoded values in test files reduce reusability. By passing variables, you can:

- Run the same tests against different environments (dev, staging, prod)
- Parameterize resource names to avoid naming conflicts
- Control test behavior from CI/CD pipelines without modifying test files
- Reuse modules tests with different input configurations

## Defining Variables in Test Files

Variables can be defined directly in `.tftest.hcl` files:

```hcl
# tests/vpc.tftest.hcl

variables {
  vpc_cidr    = "10.100.0.0/16"
  environment = "test"
  region      = "us-east-1"
}

run "create_vpc" {
  command = apply

  assert {
    condition     = aws_vpc.main.cidr_block == var.vpc_cidr
    error_message = "VPC CIDR does not match"
  }
}
```

## Overriding Variables Per Run Block

You can override variables for specific `run` blocks:

```hcl
variables {
  environment = "test"
  instance_count = 1
}

run "basic_deployment" {
  command = apply
  # Uses default variables above
}

run "scaled_deployment" {
  command = apply

  variables {
    instance_count = 3   # Override for this run only
  }

  assert {
    condition     = length(aws_instance.app) == 3
    error_message = "Expected 3 instances"
  }
}
```

## Passing Variables from the CLI

```bash
# Pass a single variable
tofu test -var="environment=staging"

# Pass multiple variables
tofu test -var="environment=staging" -var="region=us-west-2"

# Use a variable file
tofu test -var-file="staging.tfvars"
```

## Using a tfvars File for Tests

```hcl
# test.tfvars
environment    = "integration-test"
vpc_cidr       = "10.200.0.0/16"
instance_type  = "t3.micro"
name_prefix    = "tftest"
```

```bash
tofu test -var-file="test.tfvars"
```

## Environment-Specific Variables in CI/CD

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging]
    steps:
      - name: Run tests
        run: |
          tofu test \
            -var="environment=${{ matrix.environment }}" \
            -var="name_prefix=ci-${{ github.run_id }}"
```

## Dynamic Variables with UUID

Use unique prefixes to prevent resource naming conflicts between parallel test runs:

```hcl
# tests/s3.tftest.hcl
variables {
  bucket_prefix = "tftest-${uuid()}"
}

run "create_bucket" {
  command = apply

  assert {
    condition     = startswith(aws_s3_bucket.test.bucket, var.bucket_prefix)
    error_message = "Bucket name prefix mismatch"
  }
}
```

## Using Variables in Provider Configuration

```hcl
# tests/multi_region.tftest.hcl
variables {
  primary_region   = "us-east-1"
  secondary_region = "us-west-2"
}

provider "aws" {
  alias  = "primary"
  region = var.primary_region
}

provider "aws" {
  alias  = "secondary"
  region = var.secondary_region
}
```

## Variable Validation in Tests

Variables defined in the main module are automatically available in tests and their `validation` blocks are enforced:

```hcl
# variables.tf (module)
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

```hcl
# tests/invalid_env.tftest.hcl
run "test_invalid_env" {
  command = plan

  variables {
    environment = "invalid"   # Should fail validation
  }

  expect_failures = [var.environment]
}
```

## Best Practices

1. **Use `variables {}` blocks** in test files for default values; override with `-var` in CI
2. **Always prefix test resource names** with a unique identifier to avoid conflicts
3. **Use `expect_failures`** to test that invalid inputs are properly rejected
4. **Centralize common variables** in shared `.tfvars` files per environment
5. **Document required variables** for tests in a comment at the top of the test file

## Conclusion

Passing variables to OpenTofu tests makes your test suite configurable and environment-aware. By combining test-file variable defaults, CLI overrides, and CI/CD matrix strategies, you can run the same tests across multiple environments and configurations without duplicating test code.
