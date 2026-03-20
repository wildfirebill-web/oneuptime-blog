# How to Stream Logs from Cloud Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloud Backend, Logging, Terraform Cloud, Monitoring

Description: Learn how to stream and access run logs from the Terraform Cloud backend in OpenTofu, including real-time streaming, log retrieval via API, and log integration with external systems.

## Introduction

When using the cloud backend with remote execution, `tofu plan` and `tofu apply` output streams back to your terminal in real time. The same logs are also stored in Terraform Cloud and accessible via API, making it possible to integrate run logs with external monitoring systems, audit tools, and notification pipelines.

## Real-Time Log Streaming

```bash
# Logs stream automatically to terminal during remote execution
tofu plan

# Output:
# Running plan in Terraform Cloud. Output will stream here. Waiting for the plan to start...
#
# Terraform v1.7.0
# on linux_amd64
# Preparing the remote plan...
#
# Terraform used the selected providers to generate the following execution plan.
# Resource actions are indicated with the following symbols:
#   + create
#   ~ update in-place
#
# Terraform will perform the following actions:
# ...
# Plan: 2 to add, 1 to change, 0 to destroy.
#
# To perform exactly these actions, run the following command to apply:
#   tofu apply

# The run URL is printed at the end:
# Run URL: https://app.terraform.io/app/my-company/workspaces/production/runs/run-abc123
```

## Controlling Log Verbosity

```bash
# Enable debug logging for remote runs
export TF_LOG=DEBUG
tofu plan  # Shows verbose output including API calls

# Available log levels
# TRACE   - Most verbose
# DEBUG   - Detailed debugging
# INFO    - Standard information
# WARN    - Warnings only
# ERROR   - Errors only

# Log to file (useful for CI/CD)
export TF_LOG=INFO
export TF_LOG_PATH=/var/log/opentofu/plan.log
tofu apply 2>&1 | tee /tmp/apply-output.txt
```

## Retrieving Logs via API

```bash
# Get the run ID from the last plan
RUN_ID=$(curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/runs?page%5Bsize%5D=1" | \
  jq -r '.data[0].id')

echo "Run ID: $RUN_ID"

# Get plan log URL
PLAN_LOG_URL=$(curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/runs/$RUN_ID" | \
  jq -r '.data.relationships.plan.data.id')

# Get the plan object and log URL
PLAN_ID=$(curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/runs/$RUN_ID" | \
  jq -r '.data.relationships.plan.data.id')

# Retrieve plan logs
curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/plans/$PLAN_ID/log"
```

## Streaming Logs via API

```bash
#!/bin/bash
# stream-run-logs.sh - Stream logs from a running Terraform Cloud run

RUN_ID="${1:?Usage: $0 <run-id>}"

# Poll for log output while run is in progress
while true; do
  RUN_STATUS=$(curl -s \
    -H "Authorization: Bearer $TF_TOKEN" \
    "https://app.terraform.io/api/v2/runs/$RUN_ID" | \
    jq -r '.data.attributes.status')

  echo "Run status: $RUN_STATUS"

  case "$RUN_STATUS" in
    "planned"|"applied"|"errored"|"canceled"|"discarded")
      echo "Run completed with status: $RUN_STATUS"
      break
      ;;
  esac

  sleep 5
done

# Get final log
PLAN_ID=$(curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/runs/$RUN_ID" | \
  jq -r '.data.relationships.plan.data.id')

echo "=== Plan Output ==="
curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/plans/$PLAN_ID/log"
```

## GitHub Actions Log Integration

```yaml
# .github/workflows/deploy.yml
- name: OpenTofu Apply
  id: apply
  run: |
    tofu apply -auto-approve -no-color 2>&1 | tee /tmp/apply-output.txt
    echo "exit_code=${PIPESTATUS[0]}" >> $GITHUB_OUTPUT

- name: Upload apply logs as artifact
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: opentofu-apply-logs
    path: /tmp/apply-output.txt
    retention-days: 30

- name: Post apply summary
  if: always()
  run: |
    echo "## OpenTofu Apply Summary" >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
    tail -50 /tmp/apply-output.txt >> $GITHUB_STEP_SUMMARY
    echo '```' >> $GITHUB_STEP_SUMMARY
```

## Forwarding Logs to External Systems

```bash
#!/bin/bash
# forward-logs-to-splunk.sh - Forward Terraform Cloud run logs to Splunk

ORG="my-company"
SPLUNK_HEC_URL="https://splunk.internal.company.com:8088/services/collector/event"
SPLUNK_TOKEN="your-hec-token"

# Get recent runs
RUNS=$(curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/organizations/$ORG/runs?page%5Bsize%5D=10" | \
  jq -r '.data[] | @base64')

for RUN_B64 in $RUNS; do
  RUN=$(echo "$RUN_B64" | base64 -d)
  RUN_ID=$(echo "$RUN" | jq -r '.id')
  RUN_STATUS=$(echo "$RUN" | jq -r '.attributes.status')
  WORKSPACE=$(echo "$RUN" | jq -r '.relationships.workspace.data.id')
  CREATED=$(echo "$RUN" | jq -r '.attributes."created-at"')

  # Get plan log
  PLAN_ID=$(curl -s \
    -H "Authorization: Bearer $TF_TOKEN" \
    "https://app.terraform.io/api/v2/runs/$RUN_ID" | \
    jq -r '.data.relationships.plan.data.id')

  PLAN_LOG=$(curl -s \
    -H "Authorization: Bearer $TF_TOKEN" \
    "https://app.terraform.io/api/v2/plans/$PLAN_ID/log" | \
    head -c 10000)  # Limit log size

  # Forward to Splunk HEC
  curl -s -X POST \
    -H "Authorization: Splunk $SPLUNK_TOKEN" \
    -H "Content-Type: application/json" \
    "$SPLUNK_HEC_URL" \
    -d "{
      \"event\": {
        \"run_id\": \"$RUN_ID\",
        \"workspace\": \"$WORKSPACE\",
        \"status\": \"$RUN_STATUS\",
        \"created_at\": \"$CREATED\",
        \"log_excerpt\": $(echo "$PLAN_LOG" | jq -Rs .)
      },
      \"sourcetype\": \"opentofu:run\"
    }"
done
```

## Audit Log Webhook

```bash
# Terraform Cloud can send webhooks for run events
# Configure notification webhook to receive run event data

curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/notification-configurations" \
  -d '{
    "data": {
      "type": "notification-configurations",
      "attributes": {
        "destination-type": "generic",
        "enabled": true,
        "name": "Audit Log Webhook",
        "url": "https://audit-log.internal.company.com/terraform-events",
        "token": "webhook-secret-for-verification",
        "triggers": [
          "run:created",
          "run:planning",
          "run:applying",
          "run:completed",
          "run:errored"
        ]
      }
    }
  }'
```

## Conclusion

Terraform Cloud streams run logs to the local terminal automatically during remote execution. The same logs are accessible via API using the run ID, plan ID, or apply ID — enabling integration with external log management systems like Splunk, DataDog, or ELK. For CI/CD, capture output with `tee` and upload as artifacts for historical reference. Webhook notifications provide event-driven log forwarding without polling the API.
