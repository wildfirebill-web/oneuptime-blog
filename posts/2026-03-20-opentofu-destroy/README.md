# How to Use tofu destroy to Remove Infrastructure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use tofu destroy to safely remove all infrastructure managed by an OpenTofu configuration.

## Introduction

`tofu destroy` destroys all infrastructure resources managed by the current OpenTofu state. It is the reverse of `tofu apply`. By default, it shows a destruction plan and asks for confirmation. Use it to tear down development environments, clean up after testing, or decommission services.

## Basic Usage

```bash
tofu destroy

# OpenTofu shows what will be destroyed:

# Plan: 0 to add, 0 to change, 5 to destroy.
#
# Do you really want to destroy all resources?
# Only 'yes' will be accepted to confirm.
# Enter a value: yes
```

## Preview Before Destroying

```bash
# See what will be destroyed without destroying it
tofu plan -destroy
```

This is equivalent to `tofu destroy` but without execution - useful for reviewing the destruction plan.

## Auto-Approve Destroy

```bash
# Skip confirmation (CI/CD or scripted teardown)
tofu destroy -auto-approve
```

Use with caution. Only use `-auto-approve` in controlled automation contexts.

## Targeted Destroy

```bash
# Destroy specific resources without touching others
tofu destroy -target=aws_s3_bucket.old-data
tofu destroy -target=module.feature-branch
```

## Destroy with Variables

```bash
tofu destroy -var="environment=staging" -auto-approve
tofu destroy -var-file=environments/staging.tfvars -auto-approve
```

## Destroy Order

OpenTofu respects dependencies - resources are destroyed in reverse dependency order:

```text
aws_instance.web depends on aws_security_group.web
→ aws_instance.web is destroyed first
→ aws_security_group.web is destroyed after
```

You do not need to specify the order manually.

## Ephemeral Environment Teardown

```bash
#!/bin/bash
WORKSPACE="${1:-staging}"

# Select workspace and destroy all resources
tofu workspace select "$WORKSPACE"
tofu destroy -auto-approve

# Delete the workspace
tofu workspace select default
tofu workspace delete "$WORKSPACE"

echo "Workspace $WORKSPACE torn down and deleted."
```

## CI/CD Teardown on Pull Request Close

```yaml
# .github/workflows/teardown.yml
on:
  pull_request:
    types: [closed]

jobs:
  teardown:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tofu init -input=false
      - run: |
          WORKSPACE="pr-${{ github.event.pull_request.number }}"
          tofu workspace select "$WORKSPACE"
          tofu destroy -auto-approve
          tofu workspace select default
          tofu workspace delete "$WORKSPACE"
```

## What Destroy Does Not Remove

`tofu destroy` only removes resources tracked in the current state. Resources that were:
- Never managed by OpenTofu
- Removed from state with `tofu state rm`
- Created outside OpenTofu

...are not affected by `tofu destroy`.

## Protecting Critical Resources from Accidental Destroy

```hcl
resource "aws_rds_cluster" "main" {
  cluster_identifier = "production-db"

  lifecycle {
    prevent_destroy = true
  }
}
```

Attempting to destroy this resource will produce an error, protecting it from accidental deletion.

## Conclusion

`tofu destroy` is a powerful command that permanently removes infrastructure. Always use `tofu plan -destroy` first to review what will be removed. Use `prevent_destroy` lifecycle rules on critical resources like databases. For ephemeral environments, combine workspace deletion with destroy to clean up completely. In production, require manual approval even for automated destroy pipelines.
