# How to Use the -target Flag for Targeted Plans in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use the -target flag in OpenTofu to plan and apply changes to specific resources without affecting the entire configuration.

## Introduction

The `-target` flag restricts OpenTofu operations to specific resources or modules. When used, only the targeted resources and their dependencies are planned or applied. It is useful for quickly iterating on a specific resource, recovering from partial failures, or applying urgent fixes to individual components.

## Basic Usage

```bash
# Target a single resource
tofu plan -target=aws_s3_bucket.data
tofu apply -target=aws_s3_bucket.data
```

## Target a Module

```bash
# Apply all resources in a module
tofu plan -target=module.networking
tofu apply -target=module.networking
```

## Target a Resource Inside a Module

```bash
# Fully-qualified address including module path
tofu plan -target=module.networking.aws_vpc.main
tofu apply -target=module.networking.aws_vpc.main
```

## Target count Resources

```bash
# Target the first instance
tofu plan -target='aws_instance.web[0]'

# Target multiple instances
tofu plan \
  -target='aws_instance.web[0]' \
  -target='aws_instance.web[1]'
```

## Target for_each Resources

```bash
# Target a specific key
tofu plan -target='aws_s3_bucket.buckets["production"]'
```

## Multiple Targets

```bash
# Apply multiple specific resources
tofu apply \
  -target=aws_security_group.web \
  -target=aws_instance.web \
  -auto-approve
```

## Dependency Resolution

`-target` automatically includes dependencies:

```bash
# Targeting aws_instance.web also includes:
# - aws_security_group.web (if web references it)
# - aws_subnet.private (if web references it)
# etc.
```

OpenTofu resolves upstream dependencies automatically.

## When to Use -target

Appropriate use cases:
- Recovering from a partial apply failure on a specific resource
- Quickly testing a new resource in isolation
- Applying an urgent fix to one component without a full plan
- Bootstrapping a resource that others depend on

## When NOT to Use -target

```bash
# Avoid -target for routine deployments
# It leaves state in a partial state where some resources are updated
# and others are not

# Anti-pattern: always using -target in CI/CD
tofu apply -target=module.database -auto-approve  # Avoid this pattern
```

## Warning About Partial State

```bash
tofu apply -target=aws_s3_bucket.data

# Warning message:
# Warning: Applied changes may be incomplete
# The plan was created with the -target option in effect, so some actions that
# would be applied in a normal plan may have been left incomplete.
# Run another tofu plan to verify the state.
```

After a targeted apply, always run a full `tofu plan` to confirm the overall state is consistent.

## Destroy a Specific Resource

```bash
# Destroy only one resource
tofu destroy -target=aws_instance.old-web
```

## Conclusion

The `-target` flag is a tactical tool for focused operations on specific resources. Use it for recovery, rapid iteration on new resources, or urgent fixes. Always follow a targeted apply with a full `tofu plan` to verify that the overall configuration remains consistent. Avoid making `-target` a routine part of your deployment workflow — it can mask configuration drift and leave state in a partially-applied state.
