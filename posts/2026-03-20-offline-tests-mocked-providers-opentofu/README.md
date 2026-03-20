# How to Run Offline Tests with Mocked Providers in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Offline Testing, Mock Providers, CI/CD

Description: Learn how to configure OpenTofu tests that run completely offline using mocked providers, enabling fast CI pipelines and local development without cloud credentials.

## Introduction

Running infrastructure tests without cloud access is a superpower for developer productivity. With OpenTofu's mock provider feature and a few configuration tricks, you can execute a full test suite on a laptop with no internet connection-or in a restricted CI runner with no cloud credentials.

## What Makes a Test "Offline"?

An offline test:
1. Uses `mock_provider` instead of real providers.
2. Uses `override_data` for all data source lookups.
3. Does not use `terraform_remote_state` data sources that require network access.
4. Stores state locally (the default for tests).

## Complete Offline Test Setup

```hcl
# tests/offline/networking_offline.tftest.hcl

# Replace AWS provider entirely - no AWS API calls

mock_provider "aws" {
  # Provide deterministic values for resources
  mock_resource "aws_vpc" {
    defaults = {
      id                   = "vpc-offline-test-01"
      cidr_block           = "10.0.0.0/16"
      enable_dns_hostnames = true
      enable_dns_support   = true
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                = "subnet-offline-test-01"
      availability_zone = "us-east-1a"
    }
  }

  mock_resource "aws_internet_gateway" {
    defaults = {
      id = "igw-offline-test-01"
    }
  }

  # Mock data sources too
  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
      state = "available"
    }
  }
}

variables {
  vpc_cidr    = "10.0.0.0/16"
  environment = "test"
  region      = "us-east-1"
}

run "vpc_configuration_is_correct" {
  command = apply

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR should match input"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}

run "correct_number_of_subnets" {
  command = apply

  assert {
    condition     = length(aws_subnet.public) == 3
    error_message = "Should create one public subnet per AZ"
  }
}
```

## Pre-Installing Providers for Air-Gapped Environments

In fully air-gapped environments, providers must be pre-downloaded. Use the OpenTofu provider mirror:

```bash
# On a machine with internet access, mirror the providers
tofu providers mirror ./provider-cache

# In the air-gapped environment, configure the filesystem mirror
cat > ~/.tofurc <<EOF
provider_installation {
  filesystem_mirror {
    path    = "/path/to/provider-cache"
    include = ["registry.opentofu.org/*/*"]
  }
  direct {
    exclude = ["registry.opentofu.org/*/*"]
  }
}
EOF
```

## CI Pipeline Configuration for Offline Tests

```yaml
# .github/workflows/offline-tests.yml
name: Offline Tests

on: [push, pull_request]

jobs:
  offline-unit-tests:
    runs-on: ubuntu-latest
    # No cloud credentials needed
    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.7.0"

      - name: Cache providers
        uses: actions/cache@v4
        with:
          path: .terraform
          key: providers-${{ hashFiles('.terraform.lock.hcl') }}

      - name: Init (download providers once)
        run: tofu init

      - name: Run offline tests
        # No AWS_* environment variables set - tests must be fully mocked
        run: tofu test -test-directory=tests/offline
```

## Structuring Offline vs Online Tests

```text
tests/
├── offline/           ← mock_provider only; no credentials needed
│   ├── vpc.tftest.hcl
│   ├── iam.tftest.hcl
│   └── compute.tftest.hcl
└── integration/       ← real providers; requires credentials
    ├── deploy.tftest.hcl
    └── connectivity.tftest.hcl
```

## Validating That Tests Are Truly Offline

Add a CI check that deliberately unsets cloud credentials before running offline tests:

```bash
#!/usr/bin/env bash
# scripts/verify-offline-tests.sh

# Unset all AWS credential environment variables
unset AWS_ACCESS_KEY_ID
unset AWS_SECRET_ACCESS_KEY
unset AWS_SESSION_TOKEN
unset AWS_PROFILE

# Also block the metadata endpoint used by IAM role auth
export AWS_EC2_METADATA_DISABLED=true

# Run offline tests - if any use real providers, they will fail here
tofu test -test-directory=tests/offline
echo "All offline tests passed without credentials!"
```

## Conclusion

Offline tests with mocked providers are the fastest and most accessible form of infrastructure testing. By investing in a solid offline test suite, you give every developer on your team instant feedback on their changes-regardless of where or how they are working.
