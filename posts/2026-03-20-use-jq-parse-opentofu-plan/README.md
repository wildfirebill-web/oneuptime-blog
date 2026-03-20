# How to Use jq to Parse OpenTofu Plan Output

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, jq, Plan JSON, Shell Scripting, Infrastructure as Code

Description: Learn how to use jq to quickly parse, filter, and transform OpenTofu plan JSON output for shell scripts and CI/CD pipelines.

`jq` is a command-line JSON processor that pairs naturally with `tofu show -json`. It lets you extract, filter, and reshape plan data in one-liners without writing Python scripts. This guide covers the most useful `jq` patterns for working with OpenTofu plan output.

## Setup

```bash
# Generate plan JSON
tofu plan -out=tfplan
tofu show -json tfplan > plan.json

# Install jq if needed
sudo apt-get install -y jq   # Debian/Ubuntu
brew install jq               # macOS
```

## Essential jq Patterns

### Count changes by action type

```bash
# Count resources grouped by their action
jq '[.resource_changes[].change.actions[]] |
  group_by(.) |
  map({action: .[0], count: length}) |
  sort_by(.action)' plan.json
```

### List all resources being created

```bash
jq -r '.resource_changes[] |
  select(.change.actions == ["create"]) |
  .address' plan.json
```

### List all resources being destroyed

```bash
jq -r '.resource_changes[] |
  select(.change.actions == ["delete"]) |
  .address' plan.json
```

### List all destructive changes (delete or replace)

```bash
jq -r '.resource_changes[] |
  select(.change.actions | contains(["delete"])) |
  "\(.change.actions | join("+"))\t\(.address)"' plan.json
```

### Show a change summary table

```bash
jq -r '["ACTION", "TYPE", "NAME"],
  (.resource_changes[] |
    select(.change.actions != ["no-op"]) |
    [(.change.actions | join("+")), .type, .name]
  ) | @tsv' plan.json | column -t
```

## Filtering by Resource Type

```bash
# All changes to EC2 instances
jq -r '.resource_changes[] |
  select(.type == "aws_instance" and .change.actions != ["no-op"]) |
  "\(.change.actions | join("+"))\t\(.address)"' plan.json

# All IAM-related changes
jq -r '.resource_changes[] |
  select(.type | test("^aws_iam")) |
  .address' plan.json
```

## Extracting Specific Attribute Values

```bash
# Show the instance_type for all EC2 instances being created
jq -r '.resource_changes[] |
  select(.type == "aws_instance" and .change.actions == ["create"]) |
  "\(.address): \(.change.after.instance_type)"' plan.json

# Show all S3 bucket names being created
jq -r '.resource_changes[] |
  select(.type == "aws_s3_bucket" and .change.actions == ["create"]) |
  .change.after.bucket' plan.json
```

## Working with Modules

```bash
# List all changes inside a specific module
jq -r '.resource_changes[] |
  select(.module_address == "module.networking") |
  "\(.change.actions | join("+"))\t\(.address)"' plan.json

# Group changes by module
jq '.resource_changes |
  group_by(.module_address) |
  map({module: .[0].module_address, count: length})' plan.json
```

## Checking for Specific Conditions in Shell Scripts

```bash
# Exit non-zero if there are any destructive changes (useful in CI)
DESTRUCTIVE=$(jq '[.resource_changes[] |
  select(.change.actions | contains(["delete"]))] | length' plan.json)

if [[ "$DESTRUCTIVE" -gt 0 ]]; then
  echo "WARNING: $DESTRUCTIVE destructive change(s) detected."
  exit 1
fi
```

## Extracting Variable Values

```bash
# List all input variables and their values
jq -r '.variables | to_entries[] | "\(.key) = \(.value.value)"' plan.json
```

## Pretty-Printing a Specific Resource's Plan

```bash
# Show full planned state for a specific resource
jq '.resource_changes[] | select(.address == "aws_instance.web")' plan.json
```

## Conclusion

`jq` transforms OpenTofu's plan JSON into an interactive, scriptable data source. The patterns in this guide cover the most common CI/CD use cases — from counting changes and detecting destructions to extracting specific attribute values. Combine them with shell conditionals and pipeline tools to build powerful infrastructure change gates.
