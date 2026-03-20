# How to Run Specific Test Cases in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Testing, IaC, DevOps, Terraform

Description: Learn how to run specific test cases, test files, and test directories in OpenTofu using the tofu test command flags and filters.

## Introduction

When you have many tests across multiple files, running all of them for every change is slow. OpenTofu provides flags on `tofu test` to run specific files, filter by test name pattern, or target a specific test directory. This lets you run a focused subset of tests during development.

## Run All Tests

```bash
# Run all tests in the current module's tests/ directory
tofu test

# Show verbose output with each assertion result
tofu test -verbose
```

## Run a Specific Test File

```bash
# Run tests in a specific .tftest.hcl file
tofu test tests/unit.tftest.hcl

# Run multiple specific files
tofu test tests/unit.tftest.hcl tests/validation.tftest.hcl
```

## Specify the Test Directory

```bash
# Use a custom test directory (default is "tests/")
tofu test -test-directory=test

# Use tests in current directory
tofu test -test-directory=.

# Run tests from a different working directory
tofu -chdir=modules/vpc test
```

## Filter by Test Name

Use `-run` to filter tests by a regex pattern matching the run block name:

```bash
# Run only tests whose name contains "instance"
tofu test -run="instance"

# Run tests matching an exact name
tofu test -run="^creates_instance_with_correct_type$"

# Run all tests with "validation" in the name
tofu test -run="validation"

# Run tests in a specific file matching a pattern
tofu test tests/unit.tftest.hcl -run="tag"
```

## Example Test File with Named Runs

```hcl
# tests/unit.tftest.hcl

mock_provider "aws" {}

variables {
  environment   = "test"
  instance_type = "t3.micro"
}

run "creates_instance_with_correct_type" {
  command = plan
  assert {
    condition     = aws_instance.web.instance_type == "t3.micro"
    error_message = "Wrong instance type"
  }
}

run "instance_has_required_tags" {
  command = plan
  assert {
    condition     = aws_instance.web.tags["Environment"] == "test"
    error_message = "Missing Environment tag"
  }
}

run "validation_rejects_invalid_type" {
  command = plan
  variables {
    instance_type = "m5.large"
  }
  expect_failures = [var.instance_type]
}
```

```bash
# Run only tag-related tests
tofu test -run="tag"
# tests/unit.tftest.hcl... in progress
#   run "creates_instance_with_correct_type"... skip
#   run "instance_has_required_tags"... pass
#   run "validation_rejects_invalid_type"... skip

# Run only validation tests
tofu test -run="validation"
# tests/unit.tftest.hcl... in progress
#   run "creates_instance_with_correct_type"... skip
#   run "instance_has_required_tags"... skip
#   run "validation_rejects_invalid_type"... pass
```

## Passing Variables to Test Runs

```bash
# Override variables for the test run
tofu test -var="environment=staging"
tofu test -var="instance_type=t3.small" -var="region=us-west-2"

# Use a variables file
tofu test -var-file="test.tfvars"
```

## JSON Output for CI Integration

```bash
# Output results as JSON for parsing
tofu test -json

# Combined with jq for failure detection
tofu test -json | jq '.[] | select(.type == "test_run") | select(.test_run.status == "fail")'
```

## Parallel Test Execution

By default, tests within a file run sequentially (sharing state). Different test files run in parallel:

```bash
# Tests across multiple files run in parallel automatically
tofu test tests/
# unit.tftest.hcl and integration.tftest.hcl run concurrently
```

## Practical Development Workflow

```bash
# During module development: run unit tests only
tofu test tests/unit.tftest.hcl -verbose

# Test a specific feature you're working on
tofu test -run="encryption"

# Before PR: run all tests
tofu test -verbose

# In CI: run with JSON output for reporting
tofu test -json > test-results.json
```

## Conclusion

Use `-run` for filtering by name pattern, specify test files directly for file-level filtering, and use `-test-directory` to control which directory is scanned. During development, filter to just the tests relevant to your current change for fast feedback. Run all tests before committing. The `tofu test -json` flag integrates well with CI systems that parse structured output.
