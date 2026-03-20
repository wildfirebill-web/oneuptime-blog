# How to Use the Test Directory Option in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Testing, Test Directory, Infrastructure as Code, tofu test

Description: Learn how to use the test directory option in OpenTofu's native testing framework to organize test files in a dedicated directory separate from your module code.

## Introduction

OpenTofu's built-in testing framework (`tofu test`) supports a `--test-directory` flag that lets you store test files in a dedicated directory rather than alongside your module code. This keeps your module clean and separates production code from test code.

## Default Behavior

By default, `tofu test` looks for `.tftest.hcl` files in the current directory and a `tests/` subdirectory:

```
module/
├── main.tf
├── variables.tf
├── outputs.tf
└── tests/
    ├── basic.tftest.hcl
    └── advanced.tftest.hcl
```

Run tests from the module directory:

```bash
tofu test
```

## Using a Custom Test Directory

Specify a different directory with `--test-directory`:

```bash
tofu test --test-directory=integration-tests
```

Or using the short form:

```bash
tofu test -test-directory=integration-tests
```

## Recommended Directory Structure

For larger modules, organize tests by type:

```
module/
├── main.tf
├── variables.tf
├── outputs.tf
├── unit-tests/
│   ├── validation.tftest.hcl
│   └── defaults.tftest.hcl
└── integration-tests/
    ├── basic.tftest.hcl
    └── advanced.tftest.hcl
```

Run unit tests:

```bash
tofu test --test-directory=unit-tests
```

Run integration tests:

```bash
tofu test --test-directory=integration-tests
```

## Example Test File

`unit-tests/validation.tftest.hcl`:

```hcl
variables {
  environment = "dev"
  region      = "us-east-1"
}

run "validate_basic_plan" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block should be 10.0.0.0/16"
  }
}
```

## Running All Test Directories in CI

```yaml
# CI/CD pipeline
- name: Run unit tests
  run: tofu test --test-directory=unit-tests

- name: Run integration tests
  run: tofu test --test-directory=integration-tests
  if: github.ref == 'refs/heads/main'
```

## Filtering Test Files

Run only specific test files within a directory:

```bash
tofu test --filter=unit-tests/validation.tftest.hcl
```

## Conclusion

The `--test-directory` option in OpenTofu gives you flexibility to organize tests separately from module code and run different test suites for different purposes. This separation supports a clean module structure while enabling comprehensive test coverage across unit, integration, and end-to-end test categories.
