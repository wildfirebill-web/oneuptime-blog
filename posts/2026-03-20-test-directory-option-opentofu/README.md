# How to Use the -test-directory Option in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Test Directory, Infrastructure as Code, Organisation

Description: Learn how to use the `-test-directory` option in OpenTofu to specify a custom directory for test files, enabling cleaner project organisation.

## Introduction

By default, `tofu test` searches for `.tftest.hcl` and `.tofutest.hcl` files in the current working directory. The `-test-directory` option lets you point OpenTofu at a different directory, which is essential for keeping tests organised in larger modules and mono-repos.

## Basic Usage

```bash
# Run tests stored in a dedicated 'tests' subdirectory

tofu test -test-directory=tests

# Use an absolute path
tofu test -test-directory=/ci/integration-tests

# Relative path from the working directory
tofu test -test-directory=../shared-tests
```

## Recommended Directory Structures

### Flat Module Structure

For simple modules, co-locate tests in a `tests/` folder:

```text
modules/s3/
├── main.tf
├── variables.tf
├── outputs.tf
└── tests/
    ├── unit.tftest.hcl
    └── integration.tftest.hcl
```

```bash
cd modules/s3
tofu test -test-directory=tests
```

### Monorepo Structure

In a monorepo with multiple environments, centralise integration tests:

```text
infra/
├── modules/
│   ├── networking/
│   └── compute/
├── environments/
│   ├── dev/
│   └── prod/
└── tests/
    ├── networking_unit.tftest.hcl
    ├── compute_unit.tftest.hcl
    └── integration/
        ├── full_stack.tftest.hcl
        └── networking_compute.tftest.hcl
```

```bash
# Run unit tests
tofu test -test-directory=tests

# Run integration tests
tofu test -test-directory=tests/integration
```

## Setting Up a Test Directory with Fixtures

A common pattern is to have a `fixtures/` folder alongside tests for reusable variable files:

```text
tests/
├── fixtures/
│   ├── dev.tfvars
│   └── prod.tfvars
├── s3_bucket.tftest.hcl
└── iam.tftest.hcl
```

Reference fixtures from within test files:

```hcl
# tests/s3_bucket.tftest.hcl

# Use a variables file from the fixtures directory
# (OpenTofu resolves paths relative to the test file)
variables {
  bucket_name = "test-bucket-unique-12345"
  region      = "us-east-1"
}

run "basic_bucket_creation" {
  command = plan

  assert {
    condition     = aws_s3_bucket.this.bucket == "test-bucket-unique-12345"
    error_message = "Bucket name mismatch"
  }
}
```

## CI/CD Integration

Separating unit from integration tests allows different CI stages to use different directories:

```yaml
# .github/workflows/pr.yml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
      - run: tofu init
      # Fast unit tests on every PR-no cloud credentials needed
      - run: tofu test -test-directory=tests/unit

  integration-tests:
    runs-on: ubuntu-latest
    # Only run on merge to main
    if: github.ref == 'refs/heads/main'
    environment: integration
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
      - run: tofu init
      # Slower integration tests with real AWS credentials
      - run: tofu test -test-directory=tests/integration
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## Combining with `-filter`

You can use both `-test-directory` and `-filter` together. The filter path is resolved from the working directory, not from the test directory:

```bash
tofu test \
  -test-directory=tests \
  -filter=tests/s3_bucket.tftest.hcl
```

## Conclusion

The `-test-directory` option is a small but powerful flag that enables clean separation of test types and environments. Adopt a consistent directory convention early-it pays dividends as your infrastructure codebase scales.
