# How to Use the -chdir Global Option in OpenTofu - Opentofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use the -chdir global option in OpenTofu to run commands against a configuration directory without changing your shell's working directory.

## Introduction

The `-chdir` global option tells OpenTofu to switch to a specified directory before running a command. It is equivalent to running `cd /path/to/config && tofu <command>` but without actually changing the shell's working directory. This is useful for scripts that manage multiple configurations, monorepos with many Terraform roots, or CI/CD pipelines that reference configurations by path.

## Basic Usage

```bash
# Run plan against a configuration in a different directory

tofu -chdir=infrastructure/production plan

# Apply from any directory
tofu -chdir=./modules/networking apply

# Absolute path
tofu -chdir=/opt/terraform/production init
```

## Ordering Matters

`-chdir` is a global option that comes BEFORE the subcommand:

```bash
# Correct
tofu -chdir=production plan

# Incorrect - -chdir must come before the subcommand
tofu plan -chdir=production  # This will fail
```

## Monorepo Pattern

```text
infrastructure/
├── networking/
│   ├── main.tf
│   └── backend.tf
├── database/
│   ├── main.tf
│   └── backend.tf
└── applications/
    ├── main.tf
    └── backend.tf
```

```bash
# Plan all from the repo root
tofu -chdir=infrastructure/networking plan
tofu -chdir=infrastructure/database plan
tofu -chdir=infrastructure/applications plan
```

## Script Using -chdir

```bash
#!/bin/bash
# Apply all configurations from the repo root

CONFIGS=(
  "infrastructure/networking"
  "infrastructure/database"
  "infrastructure/applications"
)

for CONFIG in "${CONFIGS[@]}"; do
  echo "=== Applying: $CONFIG ==="
  tofu -chdir="$CONFIG" init -input=false
  tofu -chdir="$CONFIG" apply -auto-approve
done
```

## With Backend Configuration

```bash
# Init with backend config from repo root
tofu -chdir=infrastructure/production init \
  -backend-config=backends/production.hcl
```

## With Variable Files

```bash
# Pass var file relative to the -chdir directory
tofu -chdir=infrastructure/production apply \
  -var-file=environments/production.tfvars
```

Note: paths in `-var-file` are relative to the `-chdir` directory, not the current shell directory.

## CI/CD Usage

```yaml
# GitHub Actions: multiple environment deployments
- name: Deploy networking
  run: tofu -chdir=infrastructure/networking apply -auto-approve

- name: Deploy applications
  run: tofu -chdir=infrastructure/applications apply -auto-approve
```

## Comparison with cd

```bash
# Equivalent operations:

# Using -chdir (working directory unchanged in shell)
tofu -chdir=production plan

# Using cd (changes shell working directory)
(cd production && tofu plan)
# or
cd production
tofu plan
cd ..
```

`-chdir` is cleaner in scripts because it doesn't modify the shell's working directory, avoiding side effects for subsequent commands.

## Conclusion

`-chdir` is the clean way to run OpenTofu commands against a specific directory without changing the shell's current directory. Use it in scripts that manage multiple configuration directories, in CI/CD pipelines that run across monorepo structure, and anywhere you want to keep the script's working directory stable while targeting different configurations. Always place `-chdir` before the subcommand.
