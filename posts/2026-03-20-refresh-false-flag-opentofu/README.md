# How to Use the -refresh=false Flag in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn when and how to use the -refresh=false flag in OpenTofu to skip state refresh and speed up plan and apply operations.

## Introduction

By default, `tofu plan` and `tofu apply` refresh state by querying all managed resources from cloud APIs before computing the diff. The `-refresh=false` flag skips this refresh, using only the cached state. This can dramatically speed up operations in large configurations.

## Basic Usage

```bash
# Plan without refreshing state
tofu plan -refresh=false

# Apply without refreshing state
tofu apply -refresh=false

# Combine with auto-approve for CI
tofu apply -refresh=false -auto-approve
```

## Performance Impact

For a large configuration with 500 resources:

```
# With refresh (default):
tofu plan  # Takes 8-10 minutes (500 API calls)

# Without refresh:
tofu plan -refresh=false  # Takes 30-60 seconds (no API calls)
```

## When -refresh=false Is Safe

Use `-refresh=false` when:
- You've just run a previous operation and know state is current
- You're in a pipeline where you've just applied and immediately plan again
- During rapid iteration in development when no external changes have occurred
- Testing configuration changes against current state

```bash
# CI pipeline: safe to skip refresh after just applying
tofu apply -auto-approve
tofu plan -refresh=false  # Verify nothing else was affected
```

## When NOT to Use -refresh=false

Avoid skipping refresh when:
- Significant time has passed since the last refresh
- Resources may have been modified outside OpenTofu
- You're in a production environment about to make changes
- After incidents where manual remediation may have occurred

```bash
# Before production changes — always refresh
tofu plan  # default refresh=true
tofu apply
```

## Combining with Saved Plans

```bash
# Generate a plan without refresh (faster)
tofu plan -refresh=false -out=fast.tfplan

# Apply the plan
tofu apply fast.tfplan
```

## Refresh=false in Tests

For `tofu test`, skipping refresh speeds up test runs:

```hcl
# test/main.tftest.hcl
run "validate_config" {
  command = plan  # Default: runs without applying
  # No refresh needed in plan-mode tests
}
```

## Checking if Refresh Is Needed

```bash
# Run a refresh-only plan to see if state has drifted
tofu plan -refresh-only

# If output shows "No changes", safe to use -refresh=false
# If output shows changes, don't skip refresh
```

## Conclusion

`-refresh=false` is a performance optimization that's safe to use in specific, controlled scenarios — particularly in CI/CD pipelines immediately after an apply operation. In production environments where external changes may occur, always use the default refresh behavior to catch drift before applying changes. Never use `-refresh=false` as a permanent setting for production deployments.
