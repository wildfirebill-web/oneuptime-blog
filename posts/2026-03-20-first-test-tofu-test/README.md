# How to Write Your First Test with tofu test

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Testing, Tofu test, Infrastructure as Code, Quality Assurance

Description: Learn how to write your first infrastructure test using OpenTofu's built-in testing framework to validate resource configurations before deployment.

---

OpenTofu 1.6+ includes a native testing framework (`tofu test`) that lets you write automated tests for your infrastructure code. Tests verify that your OpenTofu configurations produce the expected resources and outputs without requiring a full deploy to validate correctness.

---

## What is tofu test?

`tofu test` is OpenTofu's built-in testing command that:
- Runs `.tftest.hcl` test files
- Creates real infrastructure, runs assertions, then destroys it
- Supports mock providers for fast unit tests
- Validates module outputs and resource attributes

---

## Project Structure

```text
my-module/
├── main.tf
├── variables.tf
├── outputs.tf
└── tests/
    ├── unit.tftest.hcl
    └── integration.tftest.hcl
```

---

## Your First Test File

```hcl
# tests/unit.tftest.hcl

# Test that the S3 bucket has the expected name and versioning

run "s3_bucket_name_is_correct" {
  command = plan  # Use 'plan' for fast unit tests

  assert {
    condition     = aws_s3_bucket.main.bucket == "my-production-bucket"
    error_message = "Bucket name does not match expected value"
  }
}

run "s3_bucket_has_versioning_enabled" {
  command = plan

  assert {
    condition     = aws_s3_bucket_versioning.main.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning must be enabled on the S3 bucket"
  }
}
```

---

## Testing a Simple Module

Suppose you have a module that creates an S3 bucket:

```hcl
# main.tf
variable "bucket_name" {}
variable "enable_versioning" { default = true }

resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name
  tags   = { ManagedBy = "opentofu" }
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}

output "bucket_arn" {
  value = aws_s3_bucket.main.arn
}
```

---

## Test File with Variables

```hcl
# tests/s3.tftest.hcl

# Set variables for all runs in this file
variables {
  bucket_name       = "test-bucket-opentofu"
  enable_versioning = true
}

run "bucket_is_created_with_correct_name" {
  command = plan

  assert {
    condition     = aws_s3_bucket.main.bucket == "test-bucket-opentofu"
    error_message = "Bucket name should be test-bucket-opentofu"
  }
}

run "versioning_is_enabled" {
  command = plan

  assert {
    condition     = aws_s3_bucket_versioning.main.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning should be Enabled"
  }
}

run "versioning_disabled_when_false" {
  command = plan

  variables {
    enable_versioning = false
  }

  assert {
    condition     = aws_s3_bucket_versioning.main.versioning_configuration[0].status == "Suspended"
    error_message = "Versioning should be Suspended when enable_versioning is false"
  }
}
```

---

## Running Tests

```bash
# Run all tests
tofu test

# Run specific test file
tofu test -filter=tests/s3.tftest.hcl

# Run specific test by name
tofu test -run=bucket_is_created_with_correct_name

# Verbose output
tofu test -verbose

# Expected output:
# tests/s3.tftest.hcl... pass
#   run "bucket_is_created_with_correct_name"... pass
#   run "versioning_is_enabled"... pass
#   run "versioning_disabled_when_false"... pass
```

---

## Integration Test (Apply + Destroy)

```hcl
# tests/integration.tftest.hcl

# Uses 'apply' command - actually creates resources
run "create_real_bucket" {
  command = apply

  variables {
    bucket_name = "tofu-test-integration-${formatdate("YYYYMMDDhhmmss", timestamp())}"
  }

  assert {
    condition     = output.bucket_arn != ""
    error_message = "Bucket ARN should not be empty after creation"
  }
}
# OpenTofu automatically destroys resources after each 'apply' test run
```

---

## Mock Providers for Offline Tests

```hcl
# tests/mocked.tftest.hcl
mock_provider "aws" {}

run "test_with_mock" {
  command = plan

  variables {
    bucket_name = "mock-test-bucket"
  }

  assert {
    condition     = aws_s3_bucket.main.bucket == "mock-test-bucket"
    error_message = "Bucket name mismatch with mock provider"
  }
}
```

---

## Best Practices

1. **Use `command = plan`** for unit tests - faster and doesn't require AWS credentials
2. **Use `command = apply`** only for integration tests that need real infrastructure
3. **Use mock providers** for truly offline tests in CI where no cloud access is available
4. **Test edge cases** - empty inputs, maximum values, and boundary conditions
5. **Run tests in CI** on every pull request before allowing merges

---

## Conclusion

`tofu test` brings infrastructure testing to the same standard as application testing. Start with `command = plan` tests to validate resource configurations quickly, and add integration tests for critical modules that need real environment validation.

---

*Deploy and monitor your tested infrastructure with [OneUptime](https://oneuptime.com).*
