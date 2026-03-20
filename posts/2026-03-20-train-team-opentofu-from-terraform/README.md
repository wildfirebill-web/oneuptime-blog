# How to Train Your Team to Use OpenTofu After Migrating from Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Team Training, Migration, Infrastructure as Code, DevOps

Description: Learn how to effectively train and onboard your engineering team to OpenTofu after migrating from Terraform, covering the key differences, new features, and workflow changes.

## Introduction

After migrating your infrastructure from Terraform to OpenTofu, the next challenge is ensuring your team is comfortable with the new tool. The good news is that OpenTofu is highly compatible with Terraform, so the learning curve is minimal for existing Terraform users.

## Day 1: Command Equivalency

Start with the most important fact: nearly every `terraform` command has a direct `tofu` equivalent.

| Terraform | OpenTofu | Notes |
|-----------|----------|-------|
| `terraform init` | `tofu init` | Identical behavior |
| `terraform plan` | `tofu plan` | Identical behavior |
| `terraform apply` | `tofu apply` | Identical behavior |
| `terraform destroy` | `tofu destroy` | Identical behavior |
| `terraform fmt` | `tofu fmt` | Identical behavior |
| `terraform validate` | `tofu validate` | Identical behavior |
| `terraform state` | `tofu state` | Identical behavior |
| `terraform test` | `tofu test` | OpenTofu has more features |

## Day 2: HCL Configuration Compatibility

Walk through existing configurations and demonstrate they work unchanged:

```bash
# Existing Terraform configurations work without modification

tofu init
tofu validate
tofu plan
```

## Day 3: OpenTofu-Specific Features

Introduce features unique to OpenTofu:

**State encryption:**
```hcl
terraform {
  encryption {
    key_provider "pbkdf2" "my_key" {
      passphrase = var.state_passphrase
    }
    method "aes_gcm" "my_method" {
      keys = key_provider.pbkdf2.my_key
    }
    state {
      method = method.aes_gcm.my_method
    }
  }
}
```

**Enhanced `tofu test` with mock providers:**
```hcl
mock_provider "aws" {
  mock_resource "aws_vpc" {
    defaults = { id = "vpc-test" }
  }
}
```

## Day 4: Workflow Changes

Update team workflows:

```yaml
# CI/CD change
# Before:
- run: terraform plan

# After:
- run: tofu plan
```

Update aliases:

```bash
# Add to ~/.zshrc or ~/.bashrc
alias terraform=tofu
```

## Day 5: Hands-On Practice

Have team members:
1. Clone the infrastructure repo
2. Run `tofu init` and `tofu plan` against a dev environment
3. Write a simple test using `tofu test`
4. Open a PR with their changes

## Key Resources

- OpenTofu documentation: `opentofu.org/docs`
- OpenTofu community Slack
- GitHub discussions at `github.com/opentofu/opentofu`

## Conclusion

Training a Terraform team to use OpenTofu is primarily about building confidence that the tools are equivalent for day-to-day work. Focus first on command equivalency and existing configuration compatibility, then introduce OpenTofu-specific features as the team grows comfortable with the migration.
