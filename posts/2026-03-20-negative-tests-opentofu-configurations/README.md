# How to Write Negative Tests for OpenTofu Configurations

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Negative Tests, Validation, Infrastructure as Code

Description: Learn how to write negative tests for OpenTofu configurations to verify that invalid inputs are properly rejected and that validation logic works as expected.

## Introduction

Negative testing verifies that your OpenTofu configurations correctly reject invalid inputs. OpenTofu's native testing framework supports `expect_failures` to assert that certain operations should fail, making it possible to test input validation, preconditions, and postconditions.

## What Are Negative Tests

Negative tests verify:
- Invalid variable values trigger validation errors.
- Preconditions reject bad configurations before resource creation.
- Postconditions catch unexpected resource states.
- Custom conditions produce the expected error messages.

## Variable Validation Tests

Given a variable with validation:

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "instance_count" {
  type = number
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}
```

Write negative tests:

```hcl
# tests/validation.tftest.hcl

run "invalid_environment_rejected" {
  command = plan

  variables {
    environment    = "production"  # Invalid - should be "prod"
    instance_count = 2
  }

  expect_failures = [
    var.environment,
  ]
}

run "instance_count_too_high" {
  command = plan

  variables {
    environment    = "dev"
    instance_count = 50  # Exceeds max of 20
  }

  expect_failures = [
    var.instance_count,
  ]
}

run "instance_count_zero" {
  command = plan

  variables {
    environment    = "dev"
    instance_count = 0  # Below minimum of 1
  }

  expect_failures = [
    var.instance_count,
  ]
}
```

## Testing Preconditions

```hcl
# main.tf
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = !startswith(var.instance_type, "t1.")
      error_message = "t1 instance types are not allowed in this module."
    }
  }
}
```

Negative test for the precondition:

```hcl
# tests/precondition.tftest.hcl

run "rejected_t1_instance_type" {
  command = plan

  variables {
    ami_id        = "ami-0abcdef1234567890"
    instance_type = "t1.micro"  # Should be rejected
  }

  expect_failures = [
    aws_instance.web,
  ]
}
```

## Testing Postconditions

```hcl
# main.tf
data "aws_ami" "app" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["app-*"]
  }

  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }
  }
}
```

```hcl
# tests/postcondition.tftest.hcl

run "arm_ami_rejected" {
  command = plan

  # Use a mock provider to return an ARM AMI
  mock_provider "aws" {
    mock_data "aws_ami" {
      defaults = {
        id           = "ami-arm64"
        architecture = "arm64"  # Should trigger postcondition failure
      }
    }
  }

  expect_failures = [
    data.aws_ami.app,
  ]
}
```

## Combining Positive and Negative Tests

A complete test file tests both valid and invalid inputs:

```hcl
# tests/full_validation.tftest.hcl

# Positive test - valid input
run "valid_configuration" {
  command = plan

  variables {
    environment    = "prod"
    instance_count = 3
  }

  assert {
    condition     = var.environment == "prod"
    error_message = "Environment should be prod"
  }
}

# Negative test - invalid environment
run "invalid_environment" {
  command = plan

  variables {
    environment    = "test"
    instance_count = 3
  }

  expect_failures = [var.environment]
}

# Negative test - instance count out of range
run "instance_count_out_of_range" {
  command = plan

  variables {
    environment    = "dev"
    instance_count = 100
  }

  expect_failures = [var.instance_count]
}
```

## Running the Tests

```bash
tofu test
tofu test -filter=tests/validation.tftest.hcl
tofu test -verbose
```

## Best Practices

- Write negative tests for every `validation` block in your variables.
- Test preconditions and postconditions with mock providers.
- Name test runs descriptively so failures are immediately understandable.
- Combine positive and negative tests in the same file for related configurations.
- Run negative tests in CI/CD on every pull request to catch regression in validation logic.

## Conclusion

Negative tests are a critical but often overlooked part of OpenTofu module quality. The `expect_failures` directive makes it straightforward to verify that your validation logic, preconditions, and postconditions correctly reject invalid configurations before they reach production.
