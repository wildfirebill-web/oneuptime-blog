# How to Use -target-file and -exclude-file in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the -target-file and -exclude-file flags in OpenTofu to manage large sets of targeted or excluded resources from files rather than command-line arguments.

## Introduction

When you need to target or exclude many resources, listing them as individual `-target` or `-exclude` flags becomes unwieldy. OpenTofu supports `-target-file` and `-exclude-file` flags that read resource addresses from files, making it easier to manage large targeted operations.

## -target-file Usage

```bash
# Create a file with resource addresses (one per line)
cat > targets.txt << 'EOF'
aws_instance.web
aws_security_group.app
module.networking.aws_vpc.main
module.database.aws_rds_instance.primary
EOF

# Use the targets file
tofu plan -target-file=targets.txt
tofu apply -target-file=targets.txt
```

## -exclude-file Usage

```bash
# Create a file of resources to exclude
cat > exclusions.txt << 'EOF'
module.monitoring
aws_cloudwatch_dashboard.main
module.legacy.aws_instance.old_server
EOF

# Apply while excluding the listed resources
tofu plan -exclude-file=exclusions.txt
tofu apply -exclude-file=exclusions.txt
```

## File Format

The file format is simple: one resource address per line, with support for comments:

```
# networking resources
module.networking.aws_vpc.main
module.networking.aws_subnet.public[0]
module.networking.aws_subnet.private[0]

# compute resources
aws_instance.web
aws_instance.api

# database
module.database.aws_db_instance.primary
```

Lines starting with `#` are treated as comments (if your version supports it, check the docs).

## Generating Target Files Dynamically

```bash
# Generate a targets file from state
tofu state list | grep "module.compute" > compute-targets.txt

# Target all compute resources
tofu plan -target-file=compute-targets.txt

# Generate based on tag or pattern
tofu state list | grep "aws_instance" > instance-targets.txt
```

## Combining with Other Flags

```bash
# Use target file with variable file
tofu apply \
  -target-file=targets.txt \
  -var-file=production.tfvars

# Use exclude file with auto-approve
tofu apply \
  -exclude-file=exclusions.txt \
  -auto-approve

# Use both target and exclude files
tofu plan \
  -target-file=targets.txt \
  -exclude-file=exclusions.txt
```

## CI/CD Use Case

```yaml
# GitHub Actions: deploy specific components
jobs:
  deploy-compute:
    runs-on: ubuntu-latest
    steps:
      - name: Generate Targets
        run: |
          cat > deploy-targets.txt << 'EOF'
          module.compute.aws_autoscaling_group.app
          module.compute.aws_launch_template.app
          module.compute.aws_lb_target_group.app
          EOF

      - name: Deploy
        run: tofu apply -target-file=deploy-targets.txt -auto-approve
```

## When to Use -target-file vs -target

Use `-target-file` when:
- Targeting more than 3-4 resources
- The target list is generated dynamically
- You want to version-control your deployment scopes
- Building reusable deployment procedures

Use `-target` when:
- Targeting 1-3 resources for a quick fix
- The targets are simple and don't change

## Conclusion

`-target-file` and `-exclude-file` make targeted OpenTofu operations more manageable at scale. By reading resource addresses from files, you can dynamically generate deployment scopes, version-control your targeting configurations, and avoid unwieldy command lines. As with all targeted operations, follow up with a full plan to ensure overall state consistency.
