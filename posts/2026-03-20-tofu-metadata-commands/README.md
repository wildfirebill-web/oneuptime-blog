# How to Use tofu metadata Commands

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use OpenTofu's tofu metadata subcommands to retrieve information about functions, schemas, and provider capabilities for tooling and integration.

## Introduction

The `tofu metadata` command group provides introspection capabilities — it lets you query information about OpenTofu functions, provider schemas, and configuration options. These commands are primarily used by tooling, editor extensions, and automation scripts rather than in day-to-day infrastructure management.

## tofu metadata functions

List all built-in OpenTofu functions:

```bash
# List all available functions
tofu metadata functions

# Output (partial):
# +------------------------+------+
# | name                   | type |
# +------------------------+------+
# | abs                    | math |
# | base64decode           | encoding |
# | base64encode           | encoding |
# | base64gzip             | encoding |
# | cidrhost               | ipnet |
# | cidrnetmask            | ipnet |
# | cidrsubnet             | ipnet |
# | coalesce               | collection |
# | ...                    | ...  |
# +------------------------+------+
```

## tofu metadata functions -json

```bash
# JSON output for machine processing
tofu metadata functions -json

# Get all function names
tofu metadata functions -json | jq '.[].name'

# Get functions by category
tofu metadata functions -json | jq '[.[] | select(.category == "string")]'

# Get function signature
tofu metadata functions -json | jq '.[] | select(.name == "cidrsubnet")'
```

## Practical Function Discovery

```bash
# Find all encoding functions
tofu metadata functions -json | jq '[.[] | select(.category == "encoding") | .name]'

# Find all network functions
tofu metadata functions -json | jq '[.[] | select(.category == "ipnet") | .name]'

# Get a function's parameters
tofu metadata functions -json | \
  jq '.[] | select(.name == "format") | {name, params: .params}'
```

## tofu metadata schemas (when available)

Retrieve provider schema information programmatically:

```bash
# Get provider schemas for providers in the configuration
tofu providers schema -json

# Get schema for a specific provider's resource
tofu providers schema -json | jq '.provider_schemas["registry.opentofu.org/hashicorp/aws"].resource_schemas["aws_instance"].attributes'
```

## Using Metadata in IDE Tooling

The `tofu metadata` commands are primarily used by:

- Language servers (terraform-ls)
- VS Code extensions
- IntelliJ plugins
- Custom tooling that needs to validate configurations

```bash
# Example: Check if a function exists
FUNC_EXISTS=$(tofu metadata functions -json | jq -r '.[] | select(.name == "yamldecode") | .name')
if [ -n "$FUNC_EXISTS" ]; then
  echo "yamldecode is available"
fi
```

## Listing All CLI Commands

```bash
# Get all available commands and subcommands
tofu --help

# Get help for a specific command
tofu metadata --help

# All metadata subcommands
tofu metadata functions --help
```

## Conclusion

The `tofu metadata` commands are designed for tooling integration and programmatic discovery of OpenTofu capabilities. Use `tofu metadata functions` to explore available built-in functions and their signatures. For most day-to-day infrastructure work, the OpenTofu documentation and IDE extensions provide better interfaces for this information, but the CLI commands are invaluable for building custom tools and automation around OpenTofu.
