# How to Handle State Drift in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, State

Description: Learn how to detect and handle state drift in OpenTofu when real infrastructure diverges from what is recorded in state.

## Introduction

State drift occurs when real infrastructure is changed outside of OpenTofu - through the cloud console, CLI, another tool, or manual API calls. OpenTofu's state no longer reflects reality, which causes incorrect plans and unexpected changes.

## Detecting Drift

```bash
# Refresh state to detect drift

tofu refresh

# Or run a plan - OpenTofu refreshes before planning by default
tofu plan

# Example drift detection in plan output:
# ~ aws_security_group.app
#   + ingress {
#       cidr_blocks      = ["10.0.0.0/8"]
#       from_port        = 8080
#       to_port          = 8080
#     }
# (someone manually added this rule via console)
```

## Drift Response Strategies

### Strategy 1: Revert Drift (Enforce Configuration)

Apply the OpenTofu configuration to restore infrastructure to the desired state:

```bash
# Preview what will change
tofu plan

# Apply to revert drift
tofu apply
```

Use when: the manual change was unintentional, violates policy, or is not the approved configuration.

### Strategy 2: Accept Drift (Update Configuration)

Update the configuration to match the manually changed infrastructure:

```hcl
# Someone added an ingress rule - add it to configuration
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = var.vpc_id

  # Add the manually-created rule to configuration
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

```bash
# Verify the plan shows no changes
tofu plan
# Expected: No changes. Your infrastructure matches the configuration.
```

### Strategy 3: Import Drifted Resources

If drift involves a new resource created outside OpenTofu:

```bash
# Import the externally-created resource
tofu import aws_s3_bucket.manually_created my-manually-created-bucket

# Then write the resource block to match
```

## Automated Drift Detection

Set up regular drift detection in CI/CD:

```yaml
# .github/workflows/drift-check.yml
name: Drift Check
on:
  schedule:
    - cron: '0 8 * * *'  # Daily at 8am

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.PLAN_ROLE_ARN }}

      - name: Check for drift
        run: |
          tofu init
          tofu plan -detailed-exitcode
          # Exit code 2 = changes detected (drift)
```

Exit codes from `tofu plan`:
- `0` = no changes
- `1` = error
- `2` = changes detected

## Preventing Drift

1. **Require OpenTofu for all changes** - disable direct console access
2. **Use SCPs / IAM policies** to prevent manual changes to managed resources
3. **Regular drift checks** - run `tofu plan` in CI daily
4. **Enforce through code review** - all infrastructure changes via PR

```hcl
# AWS Service Control Policy - prevent direct EC2 changes
resource "aws_organizations_policy" "prevent_direct_changes" {
  name    = "prevent-direct-ec2-changes"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Deny"
      Action    = ["ec2:CreateSecurityGroup", "ec2:AuthorizeSecurityGroupIngress"]
      Resource  = "*"
      Condition = {
        StringNotLike = {
          "aws:PrincipalARN" = "arn:aws:iam::*:role/TerraformRole"
        }
      }
    }]
  })
}
```

## Conclusion

State drift is inevitable in shared environments - handle it with a clear policy. Use `tofu plan` to detect drift, decide whether to revert it (apply) or accept it (update configuration), and prevent future drift through IAM restrictions and automated drift detection in CI. Regular `tofu plan` runs catch drift early before it compounds.
