# How to Test Custom Validation Rules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Validation Rules, Infrastructure as Code, Custom Validation

Description: Learn how to write tests that verify custom variable validation rules and lifecycle preconditions in OpenTofu modules behave correctly for both valid and invalid inputs.

## Introduction

OpenTofu modules often include `validation` blocks on variables to enforce naming conventions, allowed values, or format constraints. Testing these rules ensures that helpful error messages are shown when callers supply bad inputs—and that valid inputs are accepted without error.

## Defining a Variable with Validation

Here is a module variable with two validation rules:

```hcl
# modules/s3/variables.tf

variable "bucket_name" {
  type        = string
  description = "Name of the S3 bucket"

  validation {
    # Must be between 3 and 63 characters
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "Bucket name must be between 3 and 63 characters long."
  }

  validation {
    # Must be lowercase alphanumeric or hyphens only
    condition     = can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must start and end with a lowercase letter or number, and contain only lowercase letters, numbers, and hyphens."
  }
}
```

## Testing That Valid Input Passes

```hcl
# tests/validation.tftest.hcl

run "valid_bucket_name_passes" {
  command = plan

  variables {
    bucket_name = "my-valid-bucket-name"
  }

  # No expect_failures — we expect this to succeed
  assert {
    condition     = var.bucket_name == "my-valid-bucket-name"
    error_message = "Valid bucket name should be accepted without errors"
  }
}
```

## Testing That Invalid Input Fails

Use `expect_failures` to assert that a validation rule rejects bad input:

```hcl
run "rejects_bucket_name_too_short" {
  command = plan

  variables {
    # Only 2 characters — should fail the length validation
    bucket_name = "ab"
  }

  # Tell OpenTofu that we expect var.bucket_name to throw a validation error
  expect_failures = [
    var.bucket_name,
  ]
}

run "rejects_bucket_name_with_uppercase" {
  command = plan

  variables {
    bucket_name = "MyBucketWithUppercase"
  }

  expect_failures = [
    var.bucket_name,
  ]
}

run "rejects_bucket_name_starting_with_hyphen" {
  command = plan

  variables {
    bucket_name = "-invalid-start"
  }

  expect_failures = [
    var.bucket_name,
  ]
}
```

## Testing Lifecycle Preconditions

Preconditions on resources work the same way:

```hcl
# modules/ec2/main.tf
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = startswith(var.ami_id, "ami-")
      error_message = "AMI ID must start with 'ami-'."
    }
  }
}
```

```hcl
# tests/precondition.tftest.hcl

run "rejects_invalid_ami_id" {
  command = plan

  variables {
    ami_id        = "not-an-ami"
    instance_type = "t3.micro"
  }

  # Reference the resource's precondition
  expect_failures = [
    aws_instance.web,
  ]
}
```

## Testing Postconditions

```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block

  lifecycle {
    postcondition {
      condition     = self.enable_dns_support == true
      error_message = "VPC must have DNS support enabled."
    }
  }
}
```

```hcl
run "vpc_postcondition_passes" {
  command = apply

  variables {
    cidr_block = "10.0.0.0/16"
  }

  # No expect_failures — postcondition should be satisfied
  assert {
    condition     = aws_vpc.main.enable_dns_support == true
    error_message = "VPC DNS support should be enabled"
  }
}
```

## Conclusion

Testing validation rules and preconditions closes an important loop: it confirms that your module not only works with correct input but also fails loudly and helpfully with incorrect input. Pair these tests with your happy-path tests for complete module coverage.
