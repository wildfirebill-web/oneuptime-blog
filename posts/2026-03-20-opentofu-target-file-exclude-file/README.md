# How to Use -target-file and -exclude-file in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use -target-file and -exclude-file flags in OpenTofu to specify multiple resource targets or exclusions from a file instead of repeating flags on the command line.

## Introduction

When you need to target or exclude many resources, passing individual `-target` or `-exclude` flags becomes unwieldy. The `-target-file` and `-exclude-file` flags accept a file containing a list of resource addresses — one per line — making it easy to manage large sets of targets or exclusions.

## -target-file

Create a file listing resources to include:

```text
# targets.txt
aws_s3_bucket.data
aws_s3_bucket_versioning.data
module.networking
aws_iam_role.app
```

```bash
# Apply only the resources listed in targets.txt
tofu plan -target-file=targets.txt
tofu apply -target-file=targets.txt
```

This is equivalent to:

```bash
tofu apply \
  -target=aws_s3_bucket.data \
  -target=aws_s3_bucket_versioning.data \
  -target=module.networking \
  -target=aws_iam_role.app
```

## -exclude-file

Create a file listing resources to skip:

```text
# exclusions.txt
module.legacy-service
aws_cloudwatch_metric_alarm.deprecated
aws_lambda_function.old-processor
```

```bash
# Apply everything except the listed resources
tofu plan -exclude-file=exclusions.txt
tofu apply -exclude-file=exclusions.txt
```

## File Format

```text
# Lines starting with # are comments (ignored)

# Blank lines are ignored too

aws_s3_bucket.data
module.networking
aws_instance.web[0]
aws_s3_bucket.buckets["production"]
```

Supported address formats:
- Simple resources: `aws_s3_bucket.name`
- Module resources: `module.name.resource_type.resource_name`
- Count instances: `aws_instance.web[0]`
- for_each instances: `aws_s3_bucket.buckets["key"]`

## Combining with Flags

You can combine file-based and flag-based targets:

```bash
# Target resources from file PLUS additional targets from flags
tofu apply \
  -target-file=core-targets.txt \
  -target=aws_lambda_function.new-feature
```

## Use Cases

**Phased rollout: apply in stages:**

```bash
# Stage 1: networking
cat > stage1.txt << 'EOF'
module.networking
module.security-groups
EOF
tofu apply -target-file=stage1.txt

# Stage 2: compute
cat > stage2.txt << 'EOF'
module.eks
module.ec2-workers
EOF
tofu apply -target-file=stage2.txt
```

**Exclude known-failing resources:**

```bash
# Track problem resources in a file
cat > known-issues.txt << 'EOF'
# Ticket #1234 — provider bug
module.legacy-billing

# Ticket #1456 — manual configuration needed
aws_route53_record.external
EOF

tofu apply -exclude-file=known-issues.txt
```

## Generating Target Files Dynamically

```bash
# Find all resources matching a pattern in state
tofu state list | grep "module.application" > app-targets.txt

# Apply only application resources
tofu apply -target-file=app-targets.txt
```

## Warning: Partial State

Like individual `-target` and `-exclude` flags, file-based targeting can leave state partially applied. Always run a full plan after targeted operations:

```bash
tofu apply -target-file=targets.txt
tofu plan  # Verify overall state is consistent
```

## Conclusion

`-target-file` and `-exclude-file` are the ergonomic solution for managing large sets of targeted resources. Define resource lists in version-controlled text files, annotate them with comments explaining why resources are targeted or excluded, and generate lists dynamically from `tofu state list` for pattern-based targeting. Always follow targeted applies with a full plan to verify state consistency.
