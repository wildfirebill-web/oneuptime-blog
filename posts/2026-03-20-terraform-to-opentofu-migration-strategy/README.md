# How to Plan a Terraform to OpenTofu Migration Strategy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Migration, Infrastructure as Code, Strategy

Description: Learn how to plan a systematic Terraform to OpenTofu migration — assessing your current state, choosing a migration approach, handling state file compatibility, and sequencing team adoption.

## Introduction

Migrating from Terraform to OpenTofu is straightforward for most workloads because OpenTofu maintains HCL and state format compatibility. The challenge is organizational: coordinating the switch across teams, CI/CD pipelines, modules, and remote backends. A structured migration strategy minimizes risk and keeps infrastructure management uninterrupted.

## Migration Approaches

Three strategies suit different organizational contexts:

**Big Bang** — Replace all Terraform references with OpenTofu at once during a maintenance window. Suitable for small teams with few configurations.

**Parallel Run** — Run Terraform and OpenTofu side-by-side on separate configurations, progressively migrating modules. Zero-risk but slower.

**Rolling Migration** — Migrate one workspace or team at a time, establishing a verified pattern before scaling. Recommended for medium-to-large organizations.

## Pre-Migration Assessment

```hcl
# Audit your current setup before migrating
# 1. Identify all Terraform version requirements
# Check .terraform-version or required_version in configurations

# 2. List all providers and their versions
# terraform providers lock -platform=linux_amd64 -platform=darwin_amd64

# 3. Check for Terraform-exclusive features
# - Terraform Cloud/Enterprise workspaces
# - Sentinel policies (migrate to OPA)
# - Cloud-specific remote execution

# 4. Inventory all state backends
# Local, S3, Azure Blob, GCS — all compatible with OpenTofu

# 5. Check for legacy syntax
# count.index in older modules, deprecated interpolations
```

## State File Compatibility

```bash
# OpenTofu reads Terraform state files directly — no conversion needed
# State format version 4 is compatible between both tools

# Verify your state file format
terraform show -json terraform.tfstate | python3 -c "
import json,sys
state=json.load(sys.stdin)
print('Format version:', state.get('format_version'))
print('Terraform version:', state.get('terraform_version'))
"

# After migrating, OpenTofu updates the version metadata on next apply
# The state file remains readable by Terraform (backward compatible)
```

## Provider Lock File Migration

```bash
# Regenerate .terraform.lock.hcl for OpenTofu provider registry
# Delete existing lock file first
rm .terraform.lock.hcl

# Re-initialize with OpenTofu — downloads from registry.opentofu.org
tofu init

# Lock providers for multiple platforms
tofu providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64
```

## Migration Checklist

```markdown
## Pre-Migration
- [ ] Inventory all Terraform configurations and their state backends
- [ ] Check minimum Terraform version — OpenTofu supports >=1.6 feature parity
- [ ] Audit for Terraform Cloud/Enterprise-specific features
- [ ] Identify Sentinel policies to migrate to OPA
- [ ] Communicate timeline to all infrastructure teams

## Migration Steps (per configuration)
- [ ] Install OpenTofu (via tfenv, asdf, or binary)
- [ ] Test `tofu init` and `tofu plan` — compare output with `terraform plan`
- [ ] Delete `.terraform.lock.hcl` and regenerate with `tofu providers lock`
- [ ] Update CI/CD pipeline to use `tofu` binary
- [ ] Update documentation and runbooks

## Post-Migration
- [ ] Remove Terraform binary from developer workstations
- [ ] Update `.tool-versions` or `.terraform-version` files
- [ ] Archive Terraform state backups
- [ ] Validate all automated plans produce clean output
```

## Rollback Plan

```bash
# Rollback is simple — OpenTofu state files are readable by Terraform
# If issues arise, switch back to Terraform binary:

# 1. Restore Terraform lock file from git
git checkout HEAD -- .terraform.lock.hcl

# 2. Re-initialize with Terraform
terraform init

# 3. Verify plan matches expected state
terraform plan

# State files written by OpenTofu are readable by Terraform >=1.6
# No state conversion required for rollback
```

## Conclusion

Migrating from Terraform to OpenTofu requires process changes more than technical ones — the HCL syntax, state format, and provider APIs are compatible. The key steps are: replace the binary, regenerate the lock file from registry.opentofu.org, update CI/CD tooling, and migrate any Terraform Cloud/Sentinel workflows to open alternatives. Use a rolling migration to validate the pattern in one team before scaling across the organization.
