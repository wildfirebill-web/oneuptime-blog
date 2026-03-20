# How to Use .tofutest.hcl Files in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Testing, Tofutest.hcl, Infrastructure as Code, Test Files

Description: Understand the `.tofutest.hcl` file extension introduced in OpenTofu as an alternative to `.tftest.hcl` and when to use each format.

## Introduction

OpenTofu supports two test file extensions: `.tftest.hcl` and `.tofutest.hcl`. The `.tofutest.hcl` extension was introduced as an OpenTofu-specific alternative, useful when you want to clearly distinguish OpenTofu test files from Terraform test files in a shared or migrating codebase.

Both extensions are functionally identical-every feature available in `.tftest.hcl` works the same way in `.tofutest.hcl`.

## When to Use `.tofutest.hcl`

Choose `.tofutest.hcl` when:

- Your repository is in the process of migrating from Terraform to OpenTofu and you want to keep the test suites separate during transition.
- Your CI pipeline runs both `terraform test` and `tofu test`, and you need each tool to pick up only its own test files.
- Your organisation's style guide mandates OpenTofu-specific naming to signal toolchain alignment.

Choose `.tftest.hcl` when:

- You want maximum compatibility and plan to support both tools.
- You are starting fresh with OpenTofu and want familiar naming.

## Example `.tofutest.hcl` File

The syntax is identical to `.tftest.hcl`. Here is a complete example testing a networking module:

```hcl
# tests/vpc.tofutest.hcl

variables {
  vpc_cidr      = "10.0.0.0/16"
  subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  environment   = "test"
}

run "vpc_created_with_correct_cidr" {
  command = apply

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block does not match the input variable"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled on the VPC"
  }
}

run "correct_number_of_subnets_created" {
  variables {
    subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  }

  assert {
    condition     = length(aws_subnet.public) == 3
    error_message = "Expected 3 subnets, got ${length(aws_subnet.public)}"
  }
}
```

## Running `.tofutest.hcl` Files

`tofu test` automatically discovers both `.tftest.hcl` and `.tofutest.hcl` files:

```bash
# Discovers and runs both .tftest.hcl and .tofutest.hcl files

tofu test

# Run only files in a specific directory (picks up both extensions)
tofu test -test-directory=tests/
```

If you need to run only `.tofutest.hcl` files, use the `-filter` flag to match specific file names:

```bash
# Run only the vpc test file
tofu test -filter=tests/vpc.tofutest.hcl
```

## Mixing Both Extensions

You can safely mix both extensions in the same project. OpenTofu treats them identically at runtime:

```text
modules/
  networking/
    main.tf
    variables.tf
    outputs.tf
    networking.tftest.hcl       ← shared with Terraform
    networking.tofutest.hcl     ← OpenTofu-only tests (e.g., using mock providers)
```

This pattern is useful during gradual migration: keep the Terraform-compatible tests in `.tftest.hcl` and move advanced OpenTofu-specific tests (like mock providers) into `.tofutest.hcl`.

## Conclusion

The `.tofutest.hcl` extension offers a clear signal of OpenTofu alignment without sacrificing any functionality. Whether you choose `.tftest.hcl` or `.tofutest.hcl` is mostly a team convention decision-pick one and apply it consistently across your codebase.
