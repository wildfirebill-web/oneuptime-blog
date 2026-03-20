# How to Use the -json Flag for Machine-Readable Output in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Automation

Description: Learn how to use the -json flag with various OpenTofu commands to generate machine-readable JSON output for CI/CD pipelines, automation, and tooling integration.

## Introduction

The `-json` flag is available on many OpenTofu commands and outputs results in JSON format instead of human-readable text. This is essential for CI/CD pipelines, automated parsing, and building tooling around OpenTofu.

## Commands Supporting -json

### tofu plan -json

```bash
# Generate a JSON-formatted plan
tofu plan -json

# Or save a JSON plan
tofu plan -json -out=plan.json

# Parse with jq
tofu plan -json | jq '.resource_changes[] | {address: .address, action: .change.actions}'
```

JSON plan structure:
```json
{
  "format_version": "1.2",
  "terraform_version": "1.8.0",
  "resource_changes": [
    {
      "address": "aws_instance.web",
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "change": {
        "actions": ["create"],
        "before": null,
        "after": {
          "ami": "ami-0c55b159cbfafe1f0",
          "instance_type": "t3.micro"
        }
      }
    }
  ]
}
```

### tofu show -json

```bash
# Current state as JSON
tofu show -json

# Saved plan as JSON
tofu show -json plan.tfplan

# Extract resource IDs
tofu show -json | jq '.values.root_module.resources[] | {address: .address, id: .values.id}'
```

### tofu validate -json

```bash
# Validation results as JSON
tofu validate -json

# Check if valid
tofu validate -json | jq '.valid'

# Get error messages
tofu validate -json | jq '.diagnostics[] | {severity: .severity, summary: .summary}'
```

### tofu output -json

```bash
# All outputs as JSON
tofu output -json

# Specific output as JSON
tofu output -json vpc_id

# Extract value from output
tofu output -json | jq -r '.vpc_id.value'
```

### tofu version -json

```bash
# Version information as JSON
tofu version -json

# Check version
tofu version -json | jq -r '.terraform_version'
```

### tofu providers schema -json

```bash
# Provider schemas as JSON
tofu providers schema -json

# Get resource attributes
tofu providers schema -json | jq \
  '.provider_schemas["registry.opentofu.org/hashicorp/aws"].resource_schemas["aws_instance"].attributes | keys'
```

## CI/CD Integration Patterns

### Check for Planned Changes

```bash
#!/bin/bash
# check-changes.sh

tofu plan -json -out=plan.tfplan > plan_output.json

CHANGES=$(jq '.resource_changes | length' plan_output.json)
CREATES=$(jq '[.resource_changes[] | select(.change.actions[] == "create")] | length' plan_output.json)
UPDATES=$(jq '[.resource_changes[] | select(.change.actions[] == "update")] | length' plan_output.json)
DESTROYS=$(jq '[.resource_changes[] | select(.change.actions[] == "delete")] | length' plan_output.json)

echo "Total changes: $CHANGES"
echo "Creates: $CREATES, Updates: $UPDATES, Destroys: $DESTROYS"

if [ "$DESTROYS" -gt 0 ]; then
  echo "WARNING: $DESTROYS resources will be destroyed"
  jq '.resource_changes[] | select(.change.actions[] == "delete") | .address' plan_output.json
fi
```

### Post Plan Summary to PR

```bash
# Generate human-friendly summary from JSON plan
tofu show -json plan.tfplan | jq -r '
  "## OpenTofu Plan Summary\n" +
  "| Action | Count |\n" +
  "|--------|-------|\n" +
  "| Create | \([.resource_changes[] | select(.change.actions[] == "create")] | length) |\n" +
  "| Update | \([.resource_changes[] | select(.change.actions[] == "update")] | length) |\n" +
  "| Delete | \([.resource_changes[] | select(.change.actions[] == "delete")] | length) |"
'
```

## Conclusion

The `-json` flag transforms OpenTofu from an interactive tool into an automation-friendly API. Use it in CI/CD pipelines to programmatically analyze plans before applying, extract output values for downstream processes, and build custom dashboards and reporting tools. Combine with `jq` for powerful, composable data extraction from OpenTofu's structured JSON output.
