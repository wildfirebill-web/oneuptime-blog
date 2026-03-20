# How to Use State File JSON Format in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, State Management

Description: Learn how to read, parse, and understand the OpenTofu state file JSON format to extract information, troubleshoot issues, and build automation tools.

## Introduction

OpenTofu state files are stored as JSON. Understanding the structure allows you to write scripts that extract resource information, build audit tools, validate consistency, or integrate with other systems. This guide walks through the state file format and common patterns for working with it programmatically.

## The Top-Level Structure

A state file has this top-level structure:

```json
{
  "version": 4,
  "terraform_version": "1.8.0",
  "serial": 42,
  "lineage": "abc12345-6789-0000-0000-abcdef012345",
  "outputs": {
    "vpc_id": {
      "value": "vpc-0a1b2c3d",
      "type": "string"
    }
  },
  "resources": [...],
  "check_results": null
}
```

Key fields:
- `version`: The state format version (currently 4)
- `terraform_version`: The version of OpenTofu that last modified the state
- `serial`: Monotonically increasing counter; prevents state conflicts
- `lineage`: Unique ID for this state lineage; mismatches cause errors
- `outputs`: Map of output values from the root module
- `resources`: Array of all managed resources

## The Resources Array

Each entry in `resources` represents a resource block:

```json
{
  "mode": "managed",
  "type": "aws_instance",
  "name": "web",
  "provider": "provider[\"registry.opentofu.org/hashicorp/aws\"]",
  "instances": [
    {
      "schema_version": 1,
      "attributes": {
        "ami": "ami-0c55b159cbfafe1f0",
        "id": "i-0123456789abcdef0",
        "instance_type": "t3.micro",
        "tags": {
          "Name": "web-server"
        }
      },
      "sensitive_attributes": [],
      "private": "base64encodeddata=="
    }
  ]
}
```

Key fields:
- `mode`: `"managed"` for resources, `"data"` for data sources
- `type`: The resource type (e.g., `aws_instance`)
- `name`: The resource name from your configuration
- `instances`: Array of instances (for `count` > 1 or `for_each`, multiple entries)

## Extracting State Information with CLI Tools

OpenTofu provides commands to extract state information without parsing JSON directly:

```bash
# List all resources
tofu state list

# Show details for a specific resource
tofu state show aws_instance.web

# Show full state as JSON
tofu show -json > state.json

# Show outputs
tofu output -json
```

## Parsing State with jq

For scripts and automation, use `jq` to parse state JSON:

```bash
# Get all resource types and names
tofu show -json | jq '.values.root_module.resources[] | {type: .type, name: .name}'

# Get all EC2 instance IDs
tofu show -json | jq '.values.root_module.resources[] | select(.type == "aws_instance") | .values.id'

# Get resource count by type
tofu show -json | jq '[.values.root_module.resources[] | .type] | group_by(.) | map({type: .[0], count: length})'

# List all outputs
tofu output -json | jq 'to_entries[] | {name: .key, value: .value.value}'
```

## Reading State Directly

For tooling that reads the state file directly, use `tofu show -json` rather than parsing `terraform.tfstate` — it provides a stable format:

```bash
# Export state to a JSON file for external processing
tofu show -json > infrastructure-state.json

# Python script example to extract resource information
python3 << 'EOF'
import json

with open('infrastructure-state.json') as f:
    state = json.load(f)

# List all resource addresses
for resource in state.get('values', {}).get('root_module', {}).get('resources', []):
    print(f"{resource['type']}.{resource['name']}: {resource['values'].get('id', 'N/A')}")
EOF
```

## Understanding Sensitive Values in State JSON

When you run `tofu show -json`, sensitive values are redacted:

```json
{
  "type": "aws_db_instance",
  "name": "main",
  "values": {
    "password": null,
    "username": "admin"
  },
  "sensitive_values": {
    "password": true
  }
}
```

The actual raw state file still contains these values unredacted unless state encryption is enabled.

## Working with Module Resources

Resources inside modules have a different address format:

```json
{
  "module": "module.networking",
  "mode": "managed",
  "type": "aws_vpc",
  "name": "main",
  ...
}
```

```bash
# List module resources
tofu state list | grep "^module\."

# Show a module resource
tofu state show 'module.networking.aws_vpc.main'

# Extract module resources with jq
tofu show -json | jq '.values.root_module.child_modules[] | .resources[]'
```

## Conclusion

Understanding the OpenTofu state file JSON format empowers you to build automation, troubleshoot issues, and audit infrastructure. Use `tofu show -json` for a stable, documented format rather than parsing `terraform.tfstate` directly. Combined with `jq` or Python scripts, you can extract virtually any information about your managed infrastructure.
