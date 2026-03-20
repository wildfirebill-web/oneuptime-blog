# How to Set Up a Test Directory Structure in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Directory Structure, Organisation, Infrastructure as Code

Description: Learn how to organise your OpenTofu test files into a maintainable directory structure that separates unit tests, integration tests, and fixtures.

## Introduction

As your infrastructure codebase grows, test organisation becomes as important as test coverage. A well-structured test directory makes it easy to find tests, run subsets in CI, and onboard new team members. This post covers proven patterns for organising OpenTofu test files.

## Recommended Structure for a Single Module

For a standalone module with moderate complexity:

```
modules/networking/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
└── tests/
    ├── fixtures/
    │   ├── defaults.tfvars     # Minimal valid variable set
    │   ├── prod_like.tfvars    # Production-scale variables
    │   └── edge_cases.tfvars   # Boundary condition variables
    ├── unit/
    │   ├── vpc_unit.tftest.hcl
    │   └── subnet_unit.tftest.hcl
    └── integration/
        └── full_network.tftest.hcl
```

Run unit tests:
```bash
cd modules/networking
tofu init
tofu test -test-directory=tests/unit
```

## Monorepo Structure

For a platform team managing many modules:

```
infra/
├── modules/
│   ├── networking/
│   │   ├── *.tf
│   │   └── tests/
│   ├── compute/
│   │   ├── *.tf
│   │   └── tests/
│   └── database/
│       ├── *.tf
│       └── tests/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── tests/
    ├── integration/
    │   └── full_platform.tftest.hcl
    └── fixtures/
        └── shared_mocks.tftest.hcl  # Shared mock provider configs
```

## Shared Fixtures Pattern

Create a `fixtures/` directory with reusable variable files loaded by multiple test files:

```hcl
# tests/fixtures/dev_variables.tfvars
region      = "us-east-1"
environment = "dev"
vpc_cidr    = "10.0.0.0/16"
az_count    = 2
```

```bash
# Load shared fixtures in tests
tofu test -var-file=tests/fixtures/dev_variables.tfvars
```

## Naming Conventions

Adopt a consistent naming convention for test files:

| Pattern | Example | Use Case |
|---|---|---|
| `<resource>_unit.tftest.hcl` | `vpc_unit.tftest.hcl` | Mock-provider unit tests |
| `<resource>_integration.tftest.hcl` | `vpc_integration.tftest.hcl` | Real-provider tests |
| `<scenario>_e2e.tftest.hcl` | `deploy_e2e.tftest.hcl` | End-to-end workflow tests |
| `<resource>_validation.tftest.hcl` | `vpc_validation.tftest.hcl` | Input validation tests |

## CI Pipeline Structure Matching

Map your directory structure to CI jobs:

```yaml
# .github/workflows/test.yml
jobs:
  unit:
    runs-on: ubuntu-latest
    on: [pull_request]
    steps:
      - uses: opentofu/setup-opentofu@v1
      - run: tofu init && tofu test -test-directory=tests/unit

  integration:
    runs-on: ubuntu-latest
    on:
      push:
        branches: [main]
    steps:
      - uses: opentofu/setup-opentofu@v1
      - run: tofu init && tofu test -test-directory=tests/integration
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

## What Goes in Each Directory

**`tests/unit/`**: Tests using `mock_provider`, `override_resource`, or `command = plan`. No credentials required. Run on every commit.

**`tests/integration/`**: Tests using real providers that create actual cloud resources. Require credentials. Run on merge to main or nightly.

**`tests/fixtures/`**: Shared `.tfvars` files, reusable mock provider configurations, and common variable presets.

## Conclusion

A clear test directory structure is the scaffolding that makes a growing infrastructure codebase manageable. Invest in it early, document it in your CONTRIBUTING guide, and enforce it in code review—your future team members will thank you.
