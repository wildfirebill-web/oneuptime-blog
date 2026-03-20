# How to Use tofu refresh to Sync State

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, State Management

Description: Learn how to use tofu refresh and tofu apply -refresh-only to synchronize your OpenTofu state file with the actual state of your cloud infrastructure.

## Introduction

`tofu refresh` (and the preferred `tofu apply -refresh-only`) updates the state file to reflect the actual current state of your cloud infrastructure, without making any changes to the infrastructure itself. Use it when your state may be out of sync with reality.

## tofu apply -refresh-only (Preferred)

The recommended approach for refreshing state:

```bash
# Preview what refresh would change
tofu plan -refresh-only

# Apply the refresh with confirmation
tofu apply -refresh-only
```

This two-step approach shows you what changed before committing the state update.

## tofu refresh (Legacy)

```bash
# Immediately refresh state (no preview)
tofu refresh

# Note: This command is considered legacy
# Prefer: tofu apply -refresh-only
```

## Common Use Cases

### After Manual Console Changes

```bash
# Someone changed the instance type in the AWS console
# Plan shows the change:
tofu plan
# ~ aws_instance.web
#     instance_type = "t3.micro" -> "t3.small"  (detected drift)

# Accept the change by refreshing state
tofu apply -refresh-only
# State is now updated to show t3.small

# Now plan shows no drift
tofu plan
# No changes. Infrastructure is up-to-date.
```

### After Auto-Scaling Events

```bash
# Auto-scaling changed the desired count outside of OpenTofu
tofu apply -refresh-only
# Updates state to reflect current instance count
```

### Syncing Before Destructive Operations

```bash
# Always refresh before destroying to get current state
tofu apply -refresh-only
# Review what's there
tofu plan -destroy
# Proceed with informed destruction
```

## Refresh with Specific Targets

```bash
# Refresh only specific resources
tofu apply -refresh-only -target=aws_instance.web

# Refresh a module
tofu apply -refresh-only -target=module.database
```

## Refresh with Variables

```bash
# Provide variables needed by the configuration during refresh
tofu apply -refresh-only -var-file=production.tfvars
```

## Default Refresh Behavior

`tofu plan` and `tofu apply` refresh state by default:

```bash
# Refresh happens automatically (default: -refresh=true)
tofu plan

# Skip refresh for speed (when you know nothing changed externally)
tofu plan -refresh=false
```

## Understanding What Refresh Detects

After refresh, compare the state:

```bash
# Run refresh and capture output
tofu apply -refresh-only 2>&1 | tee refresh-output.txt

# Check for "Objects have changed outside of OpenTofu"
grep "Objects have changed" refresh-output.txt
```

## Refresh in CI/CD

```bash
#!/bin/bash
# daily-refresh.sh — Run via cron to detect drift

tofu plan -refresh-only -detailed-exitcode -json > refresh.json

EXIT_CODE=$?
DRIFT_COUNT=$(jq '.resource_drift | length' refresh.json)

if [ $EXIT_CODE -eq 2 ] && [ "$DRIFT_COUNT" -gt 0 ]; then
  echo "DRIFT DETECTED: $DRIFT_COUNT resources changed outside OpenTofu"
  jq '.resource_drift[].address' refresh.json
  # Send alert...
elif [ $EXIT_CODE -eq 0 ]; then
  echo "No drift detected — state is current"
else
  echo "Error during refresh"
  exit 1
fi
```

## Refresh vs Plan vs Apply

| Command | Updates State? | Changes Infrastructure? | Requires Confirmation? |
|---------|---------------|------------------------|----------------------|
| `tofu refresh` | Yes | No | No |
| `tofu apply -refresh-only` | Yes (after confirmation) | No | Yes |
| `tofu plan` | No (but reads real state) | No | No |
| `tofu apply` | Yes | Yes | Yes |

## Conclusion

Use `tofu apply -refresh-only` as the standard way to synchronize your state file with reality. The two-step approach (plan then apply) gives you visibility into what changed before committing. For automation, use the `-detailed-exitcode` flag to detect drift and trigger alerts. Regular state refresh is an important practice for maintaining accurate state in dynamic environments where changes occur outside of OpenTofu.
