# How to Use OpenTofu with Terrateam for GitHub Integration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terrateam, GitHub, GitOps, PR Automation, CI/CD

Description: Learn how to set up Terrateam for OpenTofu to enable pull request-based infrastructure deployments with plan visibility, cost estimation, and approval workflows directly in GitHub.

## Introduction

Terrateam provides GitHub-native OpenTofu automation: plan output in PR comments, cost estimates from Infracost, security scanning from tfsec, and approval gates - all driven by a simple YAML configuration file.

## GitHub App Installation and Repository Setup

```yaml
# .terrateam/config.yml

# Terrateam configuration for OpenTofu

engine:
  name: opentofu
  version: "1.7.0"

# Define directories containing OpenTofu configurations
dirs:
  environments/prod/networking:
    stacks:
      - env:
          ENVIRONMENT: prod
        hooks:
          pre_plan:
            - type: run
              cmd: echo "Planning networking..."
          post_apply:
            - type: run
              cmd: ./scripts/notify-slack.sh "Networking updated"

  environments/prod/app:
    depends_on:
      - environments/prod/networking
    stacks:
      - {}

  environments/dev:
    stacks:
      - env:
          ENVIRONMENT: dev

# Who can apply changes
when_modified:
  autoapply: false
  autoplan: true
  file_patterns:
    - "**/*.tf"
    - "**/*.tfvars"
```

## PR Workflow with Cost Estimation

```yaml
# .terrateam/config.yml
engine:
  name: opentofu
  version: "1.7.0"

integrations:
  # Enable Infracost for cost estimation
  infracost:
    enabled: true
    api_key: "${INFRACOST_API_KEY}"

  # Enable tfsec for security scanning
  tfsec:
    enabled: true
    min_severity: MEDIUM
```

## Access Control Configuration

```yaml
# .terrateam/config.yml
access_control:
  # Who can trigger plans
  plan:
    teams:
      - my-org/engineers
      - my-org/platform-team

  # Who can trigger applies
  apply:
    teams:
      - my-org/platform-team

  # Require separate person to approve vs plan
  separate_plan_and_apply: true
```

## Environment-Specific Configuration

```yaml
dirs:
  "environments/*/networking":
    tags:
      - networking

  "environments/prod/*":
    when_modified:
      autoapply: false  # Never auto-apply to prod
    hooks:
      pre_apply:
        - type: run
          cmd: |
            echo "Applying to PRODUCTION - ensure approval is received"

  "environments/dev/*":
    when_modified:
      autoapply: true  # Auto-apply dev changes
```

## Workflow Triggers via PR Comments

```bash
# Comment on PR to trigger Terrateam operations:
terrateam plan             # Plan all affected dirs
terrateam plan dir:environments/prod/networking  # Plan specific dir
terrateam apply            # Apply (if approved)
terrateam apply dir:environments/prod/app  # Apply specific dir
terrateam unlock           # Release lock
```

## Notifications Configuration

```yaml
notifications:
  - type: github_check
    # Post results as GitHub check runs

  - type: pull_request_comment
    # Post plan output as PR comments
    collapsed: true  # Collapse large outputs
```

## Conclusion

Terrateam provides a fully managed GitHub integration for OpenTofu without requiring any server infrastructure. Configuration lives in `.terrateam/config.yml` alongside your HCL code, making it easy to version-control workflow changes. The GitHub-native experience - cost estimates and security scan results directly in PR comments - makes infrastructure reviews as familiar as code reviews.
