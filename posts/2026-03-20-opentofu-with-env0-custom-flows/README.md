# How to Use OpenTofu with env0 Custom Flows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, env0, Custom Flows, CI/CD, GitOps, Workflow Automation

Description: Learn how to create custom deployment flows in env0 for OpenTofu that add approval gates, custom steps, notifications, and post-deployment validation.

## Introduction

env0 Custom Flows let you extend the standard OpenTofu plan/apply workflow with additional steps: custom approvals, integration tests, notifications, and conditional logic based on the plan output or environment type.

## Defining a Custom Flow

```yaml
# .env0/custom-flow.yml

version: 2

# Custom flow for production deployments
flows:
  - name: production-deployment
    trigger:
      - environment: production
        action: [deploy, redeploy]

    steps:
      - name: pre-plan-validation
        type: custom
        command: |
          echo "Running pre-plan validation..."
          # Check that required variables are set
          if [ -z "$TF_VAR_certificate_arn" ]; then
            echo "ERROR: certificate_arn is required for production"
            exit 1
          fi

      - name: plan
        type: tofu-plan

      - name: cost-check
        type: custom
        command: |
          # Custom cost check against budget
          COST_INCREASE=$(env0 run cost --format json | jq '.monthlyDelta')
          BUDGET=1000
          if (( $(echo "$COST_INCREASE > $BUDGET" | bc -l) )); then
            echo "Cost increase $COST_INCREASE exceeds budget $BUDGET"
            exit 1
          fi

      - name: security-scan
        type: custom
        command: |
          checkov -d . --framework terraform --quiet
          tfsec . --minimum-severity HIGH

      - name: approval
        type: approval
        approvers:
          - team: platform-team
        message: "Production deployment requires platform team approval"
        timeout_hours: 24

      - name: apply
        type: tofu-apply

      - name: post-deployment-validation
        type: custom
        command: |
          # Wait for services to be healthy
          sleep 30
          ./scripts/smoke-test.sh
```

## Environment-Specific Flow Variations

```yaml
flows:
  - name: dev-deployment
    trigger:
      - environment: development

    steps:
      - name: plan
        type: tofu-plan
      - name: apply
        type: tofu-apply
        auto_approve: true

  - name: staging-deployment
    trigger:
      - environment: staging

    steps:
      - name: plan
        type: tofu-plan
      - name: integration-tests
        type: custom
        command: ./scripts/pre-deploy-tests.sh
      - name: apply
        type: tofu-apply
```

## Custom Variables in Flows

```yaml
# Pass dynamic variables to OpenTofu based on flow context
flows:
  - name: deployment
    variables:
      - name: TF_VAR_deploy_timestamp
        value: "$(date -u +%Y%m%d%H%M%S)"
      - name: TF_VAR_git_sha
        value: "$GIT_SHA"
      - name: TF_VAR_deployer
        value: "$ENV0_DEPLOYER"
```

## Drift Remediation Flow

```yaml
flows:
  - name: drift-remediation
    trigger:
      - type: drift_detected

    steps:
      - name: notify
        type: custom
        command: |
          curl -X POST "$SLACK_WEBHOOK" \
            -H "Content-Type: application/json" \
            -d '{"text": "Drift detected in $ENV0_ENVIRONMENT_NAME!"}'

      - name: approval
        type: approval
        message: "Drift detected - approve to remediate"

      - name: apply
        type: tofu-apply
```

## Conclusion

env0 Custom Flows transform the standard OpenTofu workflow into a fully customized deployment pipeline. The multi-step approach with approval gates, security scanning, cost checks, and post-deployment validation creates a production-grade deployment process without maintaining a separate CI/CD system for infrastructure changes.
