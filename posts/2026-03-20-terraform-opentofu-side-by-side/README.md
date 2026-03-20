# How to Run Terraform and OpenTofu Side by Side

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Migration, Side-by-Side, Infrastructure as Code

Description: Learn how to run Terraform and OpenTofu side by side during migration - managing separate workloads with each tool, avoiding state conflicts, and incrementally moving configurations to OpenTofu.

## Introduction

During a migration, you may need both Terraform and OpenTofu available on the same machine and in CI. Since both tools use the same state format, the risk of accidentally running the wrong tool against a state file is real. This guide covers safe side-by-side operation.

## Installing Both Tools

```bash
# Install OpenTofu

brew install opentofu   # macOS
# or
curl -fsSL https://get.opentofu.org/install-opentofu.sh | sh

# Both tools are now available
terraform --version   # HashiCorp Terraform v1.9.0
tofu --version        # OpenTofu v1.9.0

# No conflict - different binary names
which terraform   # /usr/local/bin/terraform
which tofu        # /usr/local/bin/tofu
```

## Version Management with asdf

```bash
# Install both via asdf
asdf plugin add terraform
asdf plugin add opentofu

asdf install terraform 1.9.0
asdf install opentofu 1.9.0

# Set per-directory versions
cd legacy-terraform-project/
echo "terraform 1.9.0" > .tool-versions

cd new-opentofu-project/
echo "opentofu 1.9.0" > .tool-versions
```

## Separate Directories: Clear Ownership

The cleanest approach is clear ownership by directory:

```hcl
infrastructure/
  legacy/         ← Managed by Terraform (migrating soon)
    main.tf
    .tool-versions  # terraform 1.9.0
  networking/     ← Already migrated to OpenTofu
    main.tf
    .tool-versions  # opentofu 1.9.0
  compute/        ← Already migrated to OpenTofu
    main.tf
    .tool-versions  # opentofu 1.9.0
```

## Separate Backend Keys: Avoid State Conflicts

Use separate state keys for workloads still on Terraform vs migrated to OpenTofu:

```hcl
# legacy/backend.tf - managed by terraform
terraform {
  backend "s3" {
    bucket = "my-company-tofu-state"
    key    = "legacy/terraform.tfstate"   # "terraform" key = not yet migrated
    region = "us-east-1"
  }
}
```

```hcl
# networking/backend.tf - managed by tofu
terraform {
  backend "s3" {
    bucket = "my-company-tofu-state"
    key    = "networking/opentofu.tfstate" # "opentofu" key = migrated
    region = "us-east-1"
  }
}
```

## Warning: Don't Mix Tools on the Same State

Running both `terraform apply` and `tofu apply` against the same state file is safe (same format), but can cause lock file confusion:

```bash
# After running terraform apply:
# .terraform.lock.hcl has Terraform registry hashes

# If you then run tofu init on the same directory:
# ERROR: lock file hash mismatch (registry.opentofu.org vs registry.terraform.io)

# Solution: regenerate lock file when switching tools
rm .terraform.lock.hcl
tofu init
```

## CI/CD: Parallel Pipelines

Run separate CI jobs for Terraform and OpenTofu workloads:

```yaml
# .github/workflows/infrastructure.yml
jobs:
  # Legacy workloads still on Terraform
  terraform-legacy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.9.0"
      - run: terraform -chdir=legacy init && terraform -chdir=legacy apply -auto-approve

  # Migrated workloads on OpenTofu
  opentofu-networking:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.9.0"
      - run: tofu -chdir=networking init && tofu -chdir=networking apply -auto-approve

  opentofu-compute:
    runs-on: ubuntu-latest
    needs: opentofu-networking
    steps:
      - uses: actions/checkout@v4
      - uses: opentofu/setup-opentofu@v1
      - run: tofu -chdir=compute init && tofu -chdir=compute apply -auto-approve
```

## Migration Tracking

Track migration status in a simple file:

```markdown
# MIGRATION-STATUS.md

## OpenTofu Migration Progress

| Component | Tool | Status | Target Date |
|-----------|------|--------|-------------|
| networking | OpenTofu | ✅ Complete | 2026-01-15 |
| compute | OpenTofu | ✅ Complete | 2026-02-01 |
| databases | Terraform | 🔄 In Progress | 2026-03-01 |
| legacy-app | Terraform | 📋 Planned | 2026-04-01 |
```

## Conclusion

Running Terraform and OpenTofu side by side is safe because they share the same binary interface - just different binary names. Use separate directories with `.tool-versions` files to declare which tool manages which workload. Use different state keys to avoid accidentally running the wrong tool against a state file. Migrate component by component, tracking progress in a status document, and decommission Terraform once all workloads are migrated.
