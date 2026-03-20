# How to Use the -json Flag for Machine-Readable Output in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use the -json flag in OpenTofu commands to get machine-readable JSON output for scripting, tooling, and CI/CD integration.

## Introduction

Several OpenTofu commands support a `-json` flag that produces structured JSON output instead of human-readable text. This is essential for integrating OpenTofu into automation scripts, CI/CD pipelines, custom tooling, and dashboards that need to parse OpenTofu output programmatically.

## Commands Supporting -json

| Command | Description |
|---------|-------------|
| `tofu plan -json` | Streaming JSON plan events |
| `tofu apply -json` | Streaming JSON apply events |
| `tofu show -json` | State or plan as JSON |
| `tofu providers schema -json` | Provider schema as JSON |
| `tofu validate -json` | Validation results as JSON |
| `tofu version -json` | Version information as JSON |
| `tofu metadata functions -json` | Function signatures as JSON |

## tofu plan -json

```bash
tofu plan -json

# Each line is a JSON event:
# {"@level":"info","@message":"OpenTofu 1.6.2","@module":"terraform.ui","type":"version",...}
# {"@level":"info","@message":"aws_s3_bucket.data: Plan to create","type":"planned_change",...}
# {"@level":"info","@message":"Plan: 1 to add, 0 to change, 0 to destroy.","type":"change_summary",...}
```

Parse with jq:

```bash
# Extract only resource changes
tofu plan -json 2>/dev/null | grep '"type":"planned_change"' | jq '{action: .change.action, address: .change.resource.addr}'
```

## tofu apply -json

```bash
tofu apply -json -auto-approve

# Streaming events during apply:
# {"type":"apply_start","hook":{"resource":{"addr":"aws_s3_bucket.data"}}}
# {"type":"apply_complete","hook":{"resource":{"addr":"aws_s3_bucket.data"},"elapsed_seconds":2}}
```

## tofu show -json (State or Plan)

```bash
# State as JSON
tofu show -json

# Saved plan as JSON
tofu plan -out=tfplan
tofu show -json tfplan

# Extract all resource addresses
tofu show -json | jq '.values.root_module.resources[].address'

# Get planned changes
tofu show -json tfplan | jq '.resource_changes[] | select(.change.actions != ["no-op"]) | {address, actions: .change.actions}'
```

## tofu validate -json

```bash
tofu validate -json

# Output:
# {
#   "valid": true,
#   "error_count": 0,
#   "warning_count": 0,
#   "diagnostics": []
# }
```

```bash
# CI: fail if validation errors
tofu validate -json | jq -e '.valid'  # Exits 0 if valid, 1 if not
```

## tofu version -json

```bash
tofu version -json

# {
#   "terraform_version": "1.6.2",
#   "platform": "darwin_arm64",
#   "provider_selections": {
#     "registry.opentofu.org/hashicorp/aws": "5.31.0"
#   },
#   "terraform_outdated": false
# }
```

## Structured Plan Analysis in CI/CD

```bash
#!/bin/bash
# Analyze plan for risky changes

tofu plan -out=tfplan -input=false

PLAN_JSON=$(tofu show -json tfplan)

# Count destructive changes
DESTROYS=$(echo "$PLAN_JSON" | jq '[.resource_changes[] | select(.change.actions[] == "delete")] | length')
REPLACEMENTS=$(echo "$PLAN_JSON" | jq '[.resource_changes[] | select(.change.actions == ["delete","create"])] | length')

echo "Destroys: $DESTROYS"
echo "Replacements: $REPLACEMENTS"

if [ "$DESTROYS" -gt 0 ] || [ "$REPLACEMENTS" -gt 0 ]; then
  echo "RISKY CHANGES DETECTED — Manual approval required"
  exit 1
fi

tofu apply tfplan
```

## Posting Plan Summary to GitHub PR

```bash
# Generate human-readable plan from JSON
SUMMARY=$(tofu show -json tfplan | jq -r '
  "## Plan Summary\n" +
  "- To Add: \(.resource_changes | map(select(.change.actions == ["create"])) | length)\n" +
  "- To Change: \(.resource_changes | map(select(.change.actions == ["update"])) | length)\n" +
  "- To Destroy: \(.resource_changes | map(select(.change.actions[] == "delete")) | length)"
')

gh pr comment "$PR_NUMBER" --body "$SUMMARY"
```

## Conclusion

The `-json` flag transforms OpenTofu's human-friendly output into structured data for automation. Use `tofu show -json` for state and plan inspection, `tofu validate -json` for CI checks, `tofu plan -json` for streaming apply monitoring, and `tofu version -json` for version compatibility checks. Combine with `jq` to extract precisely the fields your tooling needs.
