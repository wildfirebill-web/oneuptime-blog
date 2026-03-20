# How to Run Tests in Parallel in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Testing, IaC, DevOps, Terraform

Description: Learn how OpenTofu runs tests in parallel across multiple test files and how to structure your test suite for optimal parallelism and isolation.

## Introduction

OpenTofu automatically runs tests from different test files in parallel, while run blocks within a single file execute sequentially. Understanding this parallelism model helps you design a test suite that is both fast and safe. Proper test isolation prevents parallel tests from interfering with each other.

## How OpenTofu Test Parallelism Works

```hcl
Test discovery: OpenTofu finds all .tftest.hcl files

File 1: unit.tftest.hcl          File 2: integration.tftest.hcl
  run "test_a" ──┐                  run "test_x" ──┐
  run "test_b" ──┤ sequential        run "test_y" ──┤ sequential
  run "test_c" ──┘                  run "test_z" ──┘

File 1 and File 2 run in parallel with each other
```

## Sequential Runs Within a File

Run blocks in a single file share state and run in order:

```hcl
# tests/integration.tftest.hcl

# These three runs are sequential - each sees state from previous runs

run "create_vpc" {
  command = apply
  # Creates vpc
}

run "create_instances" {
  command = apply
  # Can reference the VPC created in previous run
  assert {
    condition     = aws_instance.web.subnet_id != ""
    error_message = "Instance should be in the VPC created earlier"
  }
}

run "verify_connectivity" {
  command = apply
  # Validates the full deployment
}
```

## Parallel Execution Across Files

Design separate test files for parallel execution:

```text
tests/
├── unit_compute.tftest.hcl     # Tests EC2 logic
├── unit_networking.tftest.hcl  # Tests VPC logic
├── unit_storage.tftest.hcl     # Tests S3 logic
├── unit_iam.tftest.hcl         # Tests IAM logic
└── integration.tftest.hcl      # End-to-end test (sequential internally)
```

```bash
# All unit test files run in parallel
tofu test tests/unit_*.tftest.hcl
```

## Isolation for Parallel Integration Tests

For integration tests that create real resources, use unique names to prevent conflicts:

```hcl
# tests/integration_a.tftest.hcl
variables {
  # Unique prefix for this test file
  name_prefix = "test-a-${timestamp()}"
  bucket_name = "my-test-a-${random_id.suffix.hex}"
}

run "test_scenario_a" {
  command = apply
  # Uses test-a- prefix - won't conflict with integration_b tests
}
```

```hcl
# tests/integration_b.tftest.hcl
variables {
  name_prefix = "test-b-${timestamp()}"
  bucket_name = "my-test-b-${random_id.suffix.hex}"
}

run "test_scenario_b" {
  command = apply
  # Uses test-b- prefix - isolated from integration_a
}
```

## Using Separate AWS Accounts for Parallelism

For truly isolated parallel integration tests, use separate AWS accounts or regions:

```yaml
# .github/workflows/parallel-tests.yml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-file: [unit_compute, unit_networking, unit_storage]
    steps:
      - name: Run tests
        run: tofu test tests/${{ matrix.test-file }}.tftest.hcl

  integration-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        region: [us-east-1, us-west-2]
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_TEST_ROLE_ARN }}
          aws-region: ${{ matrix.region }}

      - name: Run integration tests
        run: tofu test -var="region=${{ matrix.region }}" tests/integration.tftest.hcl
```

## Avoid Shared State in Parallel Tests

Bad pattern - parallel files sharing the same resource names:

```hcl
# tests/feature_a.tftest.hcl - PROBLEMATIC if run in parallel
variables {
  bucket_name = "my-shared-test-bucket"  # ❌ Conflicts with feature_b
}
```

Good pattern - each file uses unique identifiers:

```hcl
# tests/feature_a.tftest.hcl
variables {
  bucket_name = "test-feature-a-${formatdate("YYYYMMDD-hhmmss", timestamp())}"  # ✅ Unique
}
```

## Timing Considerations

```bash
# Check total test duration
time tofu test

# Unit tests (parallel files, mocked providers): typically 5-30 seconds
# Integration tests (parallel files, real AWS): typically 2-10 minutes
```

## Conclusion

OpenTofu automatically parallelizes tests across different files, giving you free speed improvements by splitting tests into multiple files. Design your test files so that parallel execution is safe: use unique resource names for integration tests, and rely on mock providers for unit tests where there's no shared state concern. Sequential execution within a file enables multi-step integration test scenarios that depend on each other's state.
