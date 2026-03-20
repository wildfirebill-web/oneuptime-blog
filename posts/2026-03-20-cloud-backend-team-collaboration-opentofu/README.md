# How to Use Cloud Backend for Team Collaboration in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloud Backend, Team Collaboration, Terraform Cloud, Workflow

Description: Learn how to use the Terraform Cloud backend for team collaboration in OpenTofu, including plan reviews, state locking, access controls, and collaborative workflows.

## Introduction

The Terraform Cloud backend transforms OpenTofu from a local CLI tool into a collaborative platform. Teams can review plans before applying, see who ran what and when, share state safely with locking, and enforce policies. This guide covers the collaboration features and how to structure team workflows around them.

## Collaborative Plan Review Workflow

```bash
# Developer creates a plan

tofu plan -out=plan.tfplan

# Terraform Cloud shows the plan in the UI:
# https://app.terraform.io/app/my-company/workspaces/production/runs/run-abc123

# Team members can review the plan in the browser
# Comments, approvals, and discards are visible to everyone

# After approval, apply from CLI:
tofu apply plan.tfplan
# or apply from the Terraform Cloud UI
```

## Access Control and Teams

```bash
# Create teams with different permissions

# 1. Create a "developers" team (plan only)
curl -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/my-company/teams" \
  -d '{
    "data": {
      "type": "teams",
      "attributes": {
        "name": "developers",
        "organization-access": {
          "manage-workspaces": false,
          "manage-policies": false
        }
      }
    }
  }'

# 2. Grant team read+plan access to development workspaces
curl -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/team-workspaces" \
  -d '{
    "data": {
      "type": "team-workspaces",
      "attributes": {
        "access": "plan"
      },
      "relationships": {
        "team": {"data": {"type": "teams", "id": "team-devs"}},
        "workspace": {"data": {"type": "workspaces", "id": "ws-development"}}
      }
    }
  }'

# Access levels:
# read        - view state and runs
# plan        - queue plans
# write       - queue applies
# admin       - manage workspace settings
```

## State Locking for Team Safety

```bash
# Terraform Cloud automatically locks state during runs
# Multiple engineers trying to apply simultaneously:

# Engineer A:
tofu apply
# Run queued: #1 - Applying...

# Engineer B (simultaneously):
tofu apply
# Run queued: #2 - Waiting for run #1 to complete...
# Runs execute in order, no race conditions

# Force-unlock if needed (emergency only)
tofu force-unlock LOCK_ID
```

## Notification Configuration

```bash
# Configure Slack notifications for run events
curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/notification-configurations" \
  -d '{
    "data": {
      "type": "notification-configurations",
      "attributes": {
        "destination-type": "slack",
        "enabled": true,
        "name": "Slack Notifications",
        "url": "https://hooks.slack.com/services/...",
        "triggers": [
          "run:created",
          "run:planning",
          "run:needs_attention",
          "run:applying",
          "run:completed",
          "run:errored"
        ]
      }
    }
  }'
```

## Plan-Only Branches (PR Workflow)

```yaml
# .github/workflows/pr-plan.yml
# Automatically run tofu plan on pull requests

name: PR Plan

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

jobs:
  plan:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: OpenTofu Init
        run: tofu init

      - name: OpenTofu Plan
        id: plan
        run: tofu plan -no-color 2>&1 | tee /tmp/plan-output.txt
        continue-on-error: true

      - name: Comment plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('/tmp/plan-output.txt', 'utf8');
            const truncated = plan.length > 65000 ? plan.slice(-65000) + '\n...(truncated)' : plan;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## OpenTofu Plan\n```\n${truncated}\n````
            });
```

## Structured Run Workflow

```bash
# Recommended team workflow:

# 1. Developer creates feature branch
git checkout -b feature/add-database

# 2. Make infrastructure changes
# edit main.tf

# 3. Create PR - GitHub Actions runs tofu plan
# Plan output posted as PR comment

# 4. Team reviews the plan in PR comments and Terraform Cloud UI

# 5. PR is approved and merged to main

# 6. GitHub Actions runs tofu apply on main branch merge
# Plan runs automatically, apply requires manual approval in Terraform Cloud
# (if auto-apply is disabled)

# 7. Apply is approved in Terraform Cloud by authorized team member
```

## Audit Trail

```bash
# Terraform Cloud maintains a complete audit trail:
# - Who queued each run
# - Who approved/discarded each apply
# - What changes were made (plan output)
# - When each action occurred

# View run history via API
curl -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/runs?page%5Bsize%5D=20" | \
  jq '.data[] | {id: .id, status: .attributes.status, created: .attributes."created-at", by: .relationships."created-by"}'
```

## Policy Enforcement (Sentinel / OPA)

```python
# Sentinel policy example: require cost estimation approval
# policies/require-cost-estimate.sentinel

import "tfrun"
import "decimal"

# Block applies where cost increase exceeds $500/month
maximum_cost_increase = decimal.new(500)

main = rule when tfrun.phase is "apply" {
    cost_estimate = tfrun.cost_estimate
    delta = decimal.new(cost_estimate.delta_monthly_cost)
    delta <= maximum_cost_increase
}
```

```bash
# Upload policy to Terraform Cloud
curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/my-company/policies" \
  -d '{
    "data": {
      "type": "policies",
      "attributes": {
        "name": "require-cost-estimate-approval",
        "enforcement-level": "soft-mandatory"
      }
    }
  }'
```

## Conclusion

Terraform Cloud transforms OpenTofu into a collaborative tool by centralizing plan visibility, enforcing state locking, providing role-based access control, and maintaining an audit trail of all infrastructure changes. The key workflow is: developers propose changes via pull requests with plan output in PR comments, authorized team members approve applies in Terraform Cloud, and the audit trail records who approved what and when. Start with plan notifications and PR-based plan comments - these provide immediate collaboration value with minimal configuration.
