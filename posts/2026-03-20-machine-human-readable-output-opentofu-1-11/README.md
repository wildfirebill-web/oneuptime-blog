# How to Use Machine and Human Readable Output Introduced in OpenTofu 1.11

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Output Format, OpenTofu 1.11, CLI, Automation, Infrastructure as Code

Description: Learn how to use the improved machine and human readable output formats introduced in OpenTofu 1.11 for better CI/CD integration and operator experience.

## Introduction

OpenTofu 1.11 improved CLI output to better serve both human operators and automation tooling. The changes include structured JSON output improvements, consistent exit codes, and cleaner plan summaries that are easier for both people and scripts to parse.

## Human-Readable Plan Output

The improved plan output provides clearer change summaries.

```bash
# Standard plan with improved formatting
tofu plan

# Example output improvements in 1.11:
# - Change counts are more prominently displayed
# - Resource addresses are better formatted
# - Diff output is cleaner with aligned columns

OpenTofu will perform the following actions:

  # aws_s3_bucket.logs will be created
  + resource "aws_s3_bucket" "logs" {
      + bucket = "myapp-prod-logs"
      + region = "us-east-1"
    }

  # aws_db_instance.main will be updated in-place
  ~ resource "aws_db_instance" "main" {
      ~ backup_retention_period = 7 -> 14
        id                      = "myapp-prod-db"
    }

Plan: 1 to add, 1 to change, 0 to destroy.
```

## Machine-Readable JSON Output

Use `-json` for structured output that scripts can parse.

```bash
# Get plan as JSON
tofu plan -json -out=plan.tfplan 2>&1 | tee plan-log.json

# Get apply output as JSON
tofu apply -json -auto-approve 2>&1 | tee apply-log.json

# Parse plan output with jq
tofu plan -json 2>&1 | jq 'select(.type == "planned_change") | .change.resource.resource'

# Count planned changes by action
tofu plan -json 2>&1 | \
  jq 'select(.type == "planned_change") | .change.action' | \
  sort | uniq -c
```

## Parsing Apply Results in CI/CD

Extract apply results for notifications and audit logs.

```bash
#!/bin/bash
# scripts/apply-with-reporting.sh

APPLY_LOG=$(mktemp)

tofu apply -json -auto-approve 2>&1 | tee "$APPLY_LOG"

# Extract summary
ADDED=$(jq -r 'select(.type == "change_summary") | .changes.add' "$APPLY_LOG" | tail -1)
CHANGED=$(jq -r 'select(.type == "change_summary") | .changes.change' "$APPLY_LOG" | tail -1)
REMOVED=$(jq -r 'select(.type == "change_summary") | .changes.remove' "$APPLY_LOG" | tail -1)

echo "Apply complete: +${ADDED} ~${CHANGED} -${REMOVED}"

# Post to Slack
curl -s -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"OpenTofu apply complete: +${ADDED} ~${CHANGED} -${REMOVED}\"}"

rm "$APPLY_LOG"
```

## Structured Output for Monitoring

Feed JSON output into log aggregation systems.

```bash
# Send structured logs to CloudWatch
tofu apply -json 2>&1 | while read -r line; do
  echo "$line" | aws logs put-log-events \
    --log-group-name "/opentofu/applies" \
    --log-stream-name "$ENVIRONMENT-$(date +%Y%m%d)" \
    --log-events "timestamp=$(date +%s%3N),message=$line"
done
```

## Exit Codes for Automation

OpenTofu uses consistent exit codes that scripts can rely on.

```bash
# Exit codes:
# 0 = success (no changes or successful apply)
# 1 = error
# 2 = success with changes (plan with -detailed-exitcode)

tofu plan -detailed-exitcode
EXIT_CODE=$?

case $EXIT_CODE in
  0) echo "No changes needed" ;;
  1) echo "Plan failed" && exit 1 ;;
  2) echo "Changes detected, proceeding to apply" ;;
esac
```

## Output Format in GitHub Actions

Use JSON output to create PR comments with plan summaries.

```yaml
# .github/workflows/tofu-pr.yml
- name: Run OpenTofu Plan
  id: plan
  run: |
    tofu plan -json -out=plan.tfplan 2>&1 | tee plan-output.json
    ADDED=$(jq -r 'select(.type=="change_summary") | .changes.add' plan-output.json | tail -1)
    CHANGED=$(jq -r 'select(.type=="change_summary") | .changes.change' plan-output.json | tail -1)
    REMOVED=$(jq -r 'select(.type=="change_summary") | .changes.remove' plan-output.json | tail -1)
    echo "summary=Plan: +${ADDED:-0} ~${CHANGED:-0} -${REMOVED:-0}" >> $GITHUB_OUTPUT

- name: Comment on PR
  uses: actions/github-script@v7
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: `## OpenTofu Plan\n\`${{ steps.plan.outputs.summary }}\``
      })
```

## Summary

OpenTofu 1.11 improved both human-readable output for clearer operator feedback and machine-readable JSON output for better automation integration. Use `-json` flag in CI/CD pipelines to extract structured data for notifications, audit logs, and PR comments. The consistent exit codes and improved plan summaries make it easier to build reliable automation around OpenTofu operations.
