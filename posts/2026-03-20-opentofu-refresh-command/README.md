# How to Use tofu refresh to Sync State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use tofu refresh to update OpenTofu state to match the real state of your infrastructure without making any changes.

## Introduction

`tofu refresh` queries all resources in the current state and updates the state file to reflect their actual current attributes. It is a read-only operation — it never creates, updates, or destroys resources. After a refresh, `tofu plan` will use the updated state to calculate accurate diffs.

## Basic Usage

```bash
tofu refresh

# Output:
# aws_s3_bucket.data: Refreshing state... [id=acme-data-production]
# aws_instance.web: Refreshing state... [id=i-0abc123def456]
```

## When to Use Refresh

Use `tofu refresh` when:
- Resources were modified outside of OpenTofu (manual changes in console)
- You suspect drift between state and reality
- State is stale after a long period without running OpenTofu
- You want to update attribute values (like IP addresses that changed) before planning

## The Modern Alternative: -refresh-only

OpenTofu also supports a safer refresh mode as a plan:

```bash
# Show what would be refreshed without committing to state
tofu plan -refresh-only

# Apply the refresh (update state but don't make infrastructure changes)
tofu apply -refresh-only
```

The `-refresh-only` approach is preferred because it goes through the normal plan/apply workflow, shows you what changed, and lets you confirm before updating state.

## Skip Refresh for Speed

```bash
# Skip refreshing state during plan (faster for large configurations)
tofu plan -refresh=false

# Only safe when you know state is accurate
tofu apply -refresh=false
```

## Targeted Refresh

```bash
# Refresh a specific resource only
tofu apply -refresh-only -target=aws_instance.web
```

## Refresh and Then Plan

```bash
# Refresh state then plan changes
tofu refresh
tofu plan
```

Or equivalently (and more safely):

```bash
tofu plan -refresh-only   # See what changed in the real world
tofu apply -refresh-only  # Update state to match
tofu plan                 # Now plan with accurate state
```

## Detecting Drift

```bash
# The -refresh-only plan shows drift without changing infrastructure
tofu plan -refresh-only

# Output when drift exists:
# ~ aws_instance.web will be updated in-place (refreshed)
#   ~ instance_type: "t3.small" -> "t3.medium"  (changed in console)
```

## Refresh in CI/CD

```bash
# Automated drift detection
tofu plan -refresh-only -detailed-exitcode
EXIT_CODE=$?

if [ $EXIT_CODE -eq 2 ]; then
  echo "DRIFT DETECTED: State does not match infrastructure"
  # Alert team or open a ticket
elif [ $EXIT_CODE -eq 0 ]; then
  echo "No drift detected"
fi
```

## Note on tofu refresh Deprecation

While `tofu refresh` still works, the OpenTofu and Terraform communities recommend using `tofu apply -refresh-only` instead. The `-refresh-only` approach:
- Shows changes before committing them
- Goes through the plan/apply workflow
- Can be targeted with `-target`
- Works with saved plan files

## Conclusion

`tofu refresh` updates state to match reality without changing infrastructure. For new workflows, prefer `tofu plan -refresh-only` followed by `tofu apply -refresh-only` — this gives you visibility into what changed before committing the updated state. Use `tofu plan -refresh=false` when you want a fast plan and know the state is accurate.
