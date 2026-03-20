# How to Use tofu apply to Deploy Infrastructure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use tofu apply to create, update, and destroy infrastructure resources based on your OpenTofu configuration.

## Introduction

`tofu apply` executes the changes shown in a plan, creating, updating, or destroying infrastructure resources. By default, it runs a fresh plan and prompts for confirmation before making any changes. For automated pipelines, you can apply a pre-saved plan file or use `-auto-approve` to skip the prompt.

## Basic Usage

```bash
tofu apply

# OpenTofu shows the plan and asks:

# Do you want to perform these actions?
# Only 'yes' will be accepted to approve.
# Enter a value: yes
```

## Apply a Saved Plan

```bash
# Create the plan
tofu plan -out=tfplan

# Review the plan (optional)
tofu show tfplan

# Apply exactly what was planned
tofu apply tfplan
```

Using a saved plan file skips the confirmation prompt and guarantees the applied changes match what was reviewed.

## Auto-Approve (for CI/CD)

```bash
# Skip the confirmation prompt
tofu apply -auto-approve
```

Only use `-auto-approve` in automated pipelines where the plan has been reviewed in a prior step.

## Apply with Variables

```bash
tofu apply -var="environment=production"
tofu apply -var-file=environments/production.tfvars
```

## Apply Output

```bash
tofu apply

# aws_s3_bucket.data: Creating...
# aws_s3_bucket.data: Creation complete after 2s [id=acme-data-production]

# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

# Outputs:
# bucket_name = "acme-data-production"
```

## Targeted Apply

```bash
# Apply changes to specific resources only
tofu apply -target=aws_s3_bucket.data
tofu apply -target=module.networking
```

Use `-target` sparingly - it can leave state inconsistent if dependencies are not considered.

## Parallelism

```bash
# Set concurrent resource operations (default is 10)
tofu apply -parallelism=20
```

## Apply with Refresh Disabled

```bash
# Skip state refresh before applying (faster for large configs)
tofu apply -refresh=false
```

## Replace a Resource

```bash
# Force a resource to be destroyed and re-created
tofu apply -replace=aws_instance.web
```

## CI/CD Pattern: Plan then Apply

```yaml
# .github/workflows/deploy.yml
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - run: tofu init -input=false
      - run: tofu plan -out=tfplan -input=false
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan

  apply:
    needs: plan
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval in GitHub
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: tfplan
      - run: tofu apply tfplan
```

## Handling Apply Errors

```bash
# If apply partially fails, the completed resources are saved to state
# Re-run apply after fixing the error
tofu apply

# OpenTofu will pick up where it left off
```

## Conclusion

`tofu apply` is the command that makes real infrastructure changes. Always plan before applying, and use saved plan files in pipelines to ensure that exactly what was reviewed gets applied. The `-auto-approve` flag is for automation only - interactive applies should always include the confirmation prompt. After a partial failure, fix the root cause and re-run `tofu apply` - it will continue from where it left off.
