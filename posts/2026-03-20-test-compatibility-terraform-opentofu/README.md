# How to Test Compatibility Between Terraform and OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Compatibility, Testing, Migration, Infrastructure as Code

Description: Learn how to test and verify that your existing Terraform configurations are fully compatible with OpenTofu before committing to a migration.

## Introduction

OpenTofu maintains backward compatibility with Terraform configurations, but subtle differences can exist - especially with newer Terraform versions or features. Testing compatibility before migrating ensures a smooth transition without surprises.

## Side-by-Side Comparison Approach

The safest way to test compatibility is running both tools against the same configuration and comparing outputs:

```bash
# Run with Terraform

terraform init -backend=false
terraform plan -out=tf.plan
terraform show -json tf.plan > tf-plan.json

# Run with OpenTofu
tofu init -backend=false
tofu plan -out=tofu.plan
tofu show -json tofu.plan > tofu-plan.json

# Compare
diff tf-plan.json tofu-plan.json
```

## Checking Provider Compatibility

Verify all providers work with OpenTofu:

```bash
# List providers in your config
grep -r "source" . --include="*.tf" | grep "required_providers" -A20

# Initialize with OpenTofu and check for errors
tofu init 2>&1 | grep -E "(error|warning|provider)"
```

## Testing State Compatibility

Test that OpenTofu can read your existing state file without issues:

```bash
# Copy state to a test directory
cp terraform.tfstate /tmp/test-tofu/

cd /tmp/test-tofu
tofu init
tofu show   # Should display state without errors
tofu plan   # Should show no changes
```

## Validate HCL Syntax

```bash
tofu validate
```

OpenTofu's validator may flag syntax that Terraform allows or vice versa.

## Check for Unsupported Features

Some features differ between tools. Scan for potential issues:

```bash
# Check for removed attributes
grep -r "provisioner \"habitat\"" . --include="*.tf"

# Check for enterprise features
grep -r "sensitive_variables" . --include="*.tf"
grep -r "cost_estimation" . --include="*.tf"
```

## Automated Compatibility Test Script

```bash
#!/bin/bash
set -e

echo "=== Testing Terraform ==="
terraform init -reconfigure
terraform validate
terraform plan -out=tf.plan
TF_RESOURCES=$(terraform show -json tf.plan | jq '.resource_changes | length')

echo "=== Testing OpenTofu ==="
tofu init -reconfigure
tofu validate
tofu plan -out=tofu.plan
TOFU_RESOURCES=$(tofu show -json tofu.plan | jq '.resource_changes | length')

echo "Terraform resources: $TF_RESOURCES"
echo "OpenTofu resources:  $TOFU_RESOURCES"

if [ "$TF_RESOURCES" = "$TOFU_RESOURCES" ]; then
    echo "PASS: Resource counts match"
else
    echo "FAIL: Resource counts differ"
    exit 1
fi
```

## Known Differences to Watch For

- OpenTofu 1.7+ supports `tofu test` natively
- OpenTofu uses its own registry at `registry.opentofu.org`
- Some `terraform` CLI behaviors differ slightly from `tofu`
- OpenTofu adds features not in Terraform (e.g., `encrypted` state blocks)

## Conclusion

Testing compatibility before migrating from Terraform to OpenTofu reduces risk and builds confidence. A side-by-side plan comparison is the most reliable method to identify any behavioral differences before committing to the switch.
