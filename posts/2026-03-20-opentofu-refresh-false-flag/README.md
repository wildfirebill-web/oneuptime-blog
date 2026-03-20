# How to Use the -refresh=false Flag in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use the -refresh=false flag in OpenTofu to skip the state refresh step for faster plans and applies when state is known to be accurate.

## Introduction

By default, `tofu plan` and `tofu apply` refresh the state by querying each tracked resource from the cloud provider before calculating changes. For large configurations with hundreds of resources, this refresh can take minutes. The `-refresh=false` flag skips this step, making operations significantly faster when you know the state is accurate.

## Basic Usage

```bash
# Plan without refreshing state
tofu plan -refresh=false

# Apply without refreshing state
tofu apply -refresh=false
```

## How Refresh Works Normally

```bash
tofu plan
# Step 1: Read each resource from cloud APIs (REFRESH)
# aws_instance.web: Refreshing state... [id=i-0abc123]
# aws_s3_bucket.data: Refreshing state... [id=acme-data]
# ...

# Step 2: Compare refreshed state with configuration (DIFF)
# Step 3: Show plan output
```

With `-refresh=false`, step 1 is skipped.

## When It Is Safe to Use

```bash
# Safe: right after a fresh apply (state is definitely accurate)
tofu apply
tofu apply -refresh=false  # Safe — just applied, no drift possible

# Safe: in a tightly controlled automated pipeline where no manual changes occur
tofu plan -refresh=false   # Fast plan for configuration changes only
```

## When It Is Unsafe

```bash
# Unsafe: after manual changes in the cloud console
# Unsafe: after another team member made changes
# Unsafe: after cloud provider auto-scaling events
# Unsafe: if you suspect drift
```

## Performance Impact

```bash
# For a configuration with 200 resources:
time tofu plan              # Refresh 200 resources: ~3 minutes
time tofu plan -refresh=false  # Skip refresh: ~10 seconds
```

The speedup is proportional to the number of resources and API latency.

## Combining with -target

```bash
# Fast plan for a specific resource, skip full refresh
tofu plan -target=aws_lambda_function.processor -refresh=false
```

## -refresh=false in CI/CD

```bash
# In a CI/CD pipeline where state is managed carefully:
# Plan step
tofu plan -out=tfplan -refresh=false -input=false

# Apply step (uses the saved plan — no refresh needed at apply time)
tofu apply tfplan
```

When applying a saved plan file, the refresh question is moot — the plan already captured state at plan time.

## Alternative: -refresh-only for Explicit Drift Detection

```bash
# Instead of always refreshing in every plan,
# run a dedicated drift detection step
tofu plan -refresh-only

# Then in deployment runs, skip refresh
tofu plan -refresh=false -out=tfplan
tofu apply tfplan
```

## The Default Behavior

```bash
# Default: refresh is enabled
tofu plan          # Equivalent to:
tofu plan -refresh=true
```

## Post-Apply Verify

```bash
# After apply, verify state is clean without refreshing
tofu plan -refresh=false
# Should show "No changes. Infrastructure is up-to-date."
```

## Conclusion

`-refresh=false` is a performance optimization for environments where state is known to be accurate. Use it in CI/CD pipelines with controlled infrastructure, after fresh applies, or when testing configuration changes where state drift is not a concern. For drift detection, use `tofu plan -refresh-only` as a dedicated step rather than embedding refresh in every deployment plan. Never use `-refresh=false` after manual cloud console changes.
