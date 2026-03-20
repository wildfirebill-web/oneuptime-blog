# How to Use the -exclude Flag in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the OpenTofu -exclude flag to skip specific resources during plan and apply operations, the inverse of the -target flag.

## Introduction

The `-exclude` flag (available in OpenTofu 1.9+) is the inverse of `-target`. While `-target` limits operations to specific resources, `-exclude` applies to everything EXCEPT the specified resources. This is useful when you want to apply most of your configuration but skip certain problematic resources.

## Basic Usage

```bash
# Exclude a specific resource from the plan

tofu plan -exclude=aws_instance.web

# Exclude multiple resources
tofu plan \
  -exclude=aws_instance.web \
  -exclude=aws_rds_instance.database

# Exclude a module
tofu plan -exclude=module.monitoring

# Apply while excluding specific resources
tofu apply -exclude=module.monitoring
```

## Use Cases

### Skipping a Broken Resource

When one resource is failing and blocking the entire apply:

```bash
# Apply everything except the problematic resource
tofu apply -exclude=aws_lambda_function.flaky_function

# Fix the issue, then apply the excluded resource
tofu apply -target=aws_lambda_function.flaky_function
```

### Gradual Migrations

Apply most of the configuration while migrating a specific component:

```bash
# Apply everything except the old database (being migrated separately)
tofu apply -exclude=aws_db_instance.legacy_database
```

### Excluding Third-Party Resources

When some resources are managed by another team:

```bash
# Don't touch the shared security groups
tofu apply -exclude=module.shared_security_groups
```

## Syntax for Different Resource Types

```bash
# Single resource
-exclude=aws_instance.web

# Indexed resource (count)
-exclude='aws_instance.web[0]'

# for_each resource
-exclude='aws_instance.web["prod"]'

# Module
-exclude=module.networking

# Resource within a module
-exclude=module.networking.aws_subnet.public
```

## -exclude vs -target Comparison

| Aspect | -target | -exclude |
|--------|---------|----------|
| Applies to | ONLY specified resources | Everything EXCEPT specified resources |
| Use when | Isolating specific resources | Skipping specific resources |
| Blast radius | Small (targeted) | Large (almost everything) |

## Combining with -target

You can combine `-target` and `-exclude` but it can be confusing - use one or the other:

```bash
# Usually avoid combining - pick one approach
# -target is more explicit for isolation
# -exclude is better for "skip this one thing"
```

## Follow Up with Full Plan

After using `-exclude`, run a full plan to verify consistency:

```bash
tofu apply -exclude=module.monitoring

# Verify state is consistent
tofu plan
# Should only show the excluded resources as out of sync (if applicable)
```

## Conclusion

The `-exclude` flag is OpenTofu's complement to `-target`, offering a "skip this" rather than "only this" approach. Use it when you want to apply most of your configuration but need to temporarily skip a specific resource. Like `-target`, it's an escape hatch rather than a routine workflow tool - always follow up with a full plan to ensure state consistency.
