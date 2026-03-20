# How to Use Terragrunt run-all for Multi-Module Operations with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terragrunt, run-all, Multi-Module, Orchestration

Description: Learn how to use Terragrunt's run-all command to apply, plan, and destroy multiple OpenTofu modules simultaneously while respecting dependency order.

## Introduction

Terragrunt's `run-all` command discovers all `terragrunt.hcl` files beneath a directory and runs the specified OpenTofu command on each one in dependency order. This enables applying an entire environment with a single command rather than visiting each module directory individually.

## Basic run-all Usage

```bash
# Plan all modules under environments/prod
terragrunt run-all plan --terragrunt-working-dir environments/prod

# Apply all modules (prompts for confirmation)
terragrunt run-all apply --terragrunt-working-dir environments/prod

# Destroy all modules in reverse dependency order
terragrunt run-all destroy --terragrunt-working-dir environments/prod

# Run from within the directory
cd environments/prod
terragrunt run-all apply
```

## Dependency-Aware Execution Order

Given this directory structure:

```
environments/prod/
├── networking/           # no dependencies
├── database/             # depends on networking
├── cache/                # depends on networking
└── services/
    ├── api/              # depends on networking, database, cache
    └── worker/           # depends on networking, database, cache
```

Terragrunt automatically determines the correct apply order:

```
1. networking (parallelizable - no deps)
2. database, cache (parallelizable - both depend only on networking)
3. api, worker (parallelizable - after all their deps)
```

## Controlling Parallelism

```bash
# Run with limited parallelism (default is unlimited)
terragrunt run-all apply --terragrunt-parallelism 3

# Run sequentially (parallelism = 1)
terragrunt run-all apply --terragrunt-parallelism 1
```

## Excluding Specific Modules

```bash
# Exclude specific modules from run-all
terragrunt run-all apply \
  --terragrunt-exclude-dir environments/prod/services/worker \
  --terragrunt-exclude-dir environments/prod/cache
```

Or use the `terragrunt.hcl` skip configuration:

```hcl
# environments/prod/legacy/terragrunt.hcl
# Skip this module during run-all operations
terraform {
  source = "../../../modules/legacy"
}

# The skip attribute prevents run-all from touching this module
skip = true
```

## Running on a Subset of Modules

```bash
# Only run modules in the services subtree
terragrunt run-all apply --terragrunt-working-dir environments/prod/services

# Run a specific command on all modules
terragrunt run-all output --terragrunt-working-dir environments/prod
```

## Non-Interactive Mode for CI/CD

```bash
# Auto-approve for CI/CD pipelines
terragrunt run-all apply \
  --terragrunt-non-interactive \
  --terragrunt-working-dir environments/prod

# With OpenTofu auto-approve flag
terragrunt run-all apply \
  --terragrunt-non-interactive \
  -auto-approve \
  --terragrunt-working-dir environments/prod
```

## Handling Failures

```bash
# Continue applying other modules even if one fails
terragrunt run-all apply --terragrunt-ignore-dependency-errors

# Stop on first error (default behavior)
terragrunt run-all apply  # stops when any module fails
```

## run-all with Custom Commands

```bash
# Run tofu init on all modules
terragrunt run-all init --terragrunt-working-dir environments/prod

# Run validate on all modules
terragrunt run-all validate --terragrunt-working-dir environments/prod

# Format check all modules
terragrunt run-all fmt --check --terragrunt-working-dir .
```

## Viewing the Execution Plan

```bash
# See which modules would be affected and in what order
terragrunt run-all plan --terragrunt-working-dir environments/prod \
  2>&1 | grep -E "Module|Executing"

# Output dependency graph
terragrunt graph-dependencies --terragrunt-working-dir environments/prod
```

## CI/CD Pipeline Integration

```yaml
# .github/workflows/deploy.yml
- name: Plan all modules
  run: |
    terragrunt run-all plan \
      --terragrunt-working-dir environments/${{ github.base_ref }} \
      --terragrunt-non-interactive \
      -out=tfplan

- name: Apply on merge
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  run: |
    terragrunt run-all apply \
      --terragrunt-working-dir environments/prod \
      --terragrunt-non-interactive \
      -auto-approve
```

## Conclusion

Terragrunt's `run-all` command turns multi-module operations from a manual, error-prone process into a single declarative command. The automatic dependency resolution ensures modules are applied in the correct order, parallelism speeds up deployments, and the `--terragrunt-exclude-dir` flag gives fine-grained control when you need to skip specific modules.
