# How to Use the -exclude Flag in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use the -exclude flag in OpenTofu to skip specific resources or modules during plan and apply operations.

## Introduction

The `-exclude` flag is the inverse of `-target`. Instead of specifying what to include, it specifies what to skip. OpenTofu processes everything in the configuration except the excluded resources and their dependents. This is useful when a specific resource is failing or temporarily out of scope but you still want to apply the rest of the configuration.

## Basic Usage

```bash
# Exclude a specific resource from the plan

tofu plan -exclude=aws_instance.web

# Exclude a resource from apply
tofu apply -exclude=aws_instance.web
```

## Exclude a Module

```bash
# Skip an entire module during apply
tofu plan -exclude=module.database
tofu apply -exclude=module.database
```

## Exclude Multiple Resources

```bash
# Exclude multiple resources
tofu apply \
  -exclude=module.database \
  -exclude=aws_route53_record.api \
  -auto-approve
```

## Exclude vs Target

```bash
# -target: apply ONLY these resources
tofu apply -target=module.networking

# -exclude: apply EVERYTHING EXCEPT these resources
tofu apply -exclude=module.database
```

Use `-exclude` when you want to apply almost everything but skip specific problematic resources.

## Common Use Case: Skip a Failing Resource

```bash
# If aws_lambda_function.processor is failing due to a temporary issue,
# apply everything else and revisit it later
tofu apply -exclude=aws_lambda_function.processor -auto-approve
```

## Exclude a count Resource

```bash
# Skip a specific instance
tofu plan -exclude='aws_instance.web[2]'
```

## Exclude a for_each Resource

```bash
# Skip a specific key
tofu plan -exclude='aws_s3_bucket.buckets["legacy"]'
```

## Warning About Partial State

Like `-target`, `-exclude` can leave state partially applied:

```bash
tofu apply -exclude=module.database

# Warning message:
# Warning: Applied changes may be incomplete
# The plan was created with the -exclude option in effect, so some actions that
# would be applied in a normal plan may have been left incomplete.
```

After an excluded apply, run a full `tofu plan` to verify the overall state.

## Dependency Considerations

When you exclude a resource, any resources that depend on the excluded one are also implicitly skipped:

```bash
# If aws_vpc.main is excluded, all subnets and instances that
# reference it will also be excluded automatically
tofu plan -exclude=aws_vpc.main
```

## Temporary Workaround Pattern

```bash
# Known issue with a module in staging - skip it for now
tofu apply \
  -exclude=module.legacy-service \
  -var-file=environments/staging.tfvars \
  -auto-approve

# Track the skipped module
echo "TODO: module.legacy-service was excluded on $(date) due to issue #1234" >> KNOWN_ISSUES.txt
```

## Difference from -refresh=false

```bash
# -exclude skips specific resources entirely (no plan, no apply)
tofu apply -exclude=module.database

# -refresh=false skips the state refresh step for ALL resources
tofu apply -refresh=false
```

## Conclusion

The `-exclude` flag lets you skip specific resources while applying the rest of the configuration. Use it when a known-failing resource is blocking deployment of unrelated changes, or when a module needs to be temporarily bypassed. Like `-target`, `-exclude` creates partial state - always follow up with a full plan to verify consistency. Track excluded resources in a comment or issue so they are not forgotten.
