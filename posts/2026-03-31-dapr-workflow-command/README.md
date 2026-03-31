# How to Use the dapr workflow Command

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CLI, Workflow, Orchestration, Management

Description: Learn how to use the dapr workflow command to start, terminate, pause, resume, and purge Dapr workflow instances from the CLI.

---

## Overview

The `dapr workflow` command provides lifecycle management for Dapr workflow instances directly from the command line. You can start new workflow instances, check their status, pause and resume running workflows, terminate stuck instances, and purge completed workflow history.

## Starting a Workflow Instance

```bash
dapr workflow run \
  --app-id order-processor \
  --workflow-component dapr \
  --workflow-name OrderFulfillmentWorkflow \
  --input '{"orderId": "ord-123", "customerId": "cust-456"}'
```

The command returns the workflow instance ID:

```
Successfully started workflow. Instance ID: abc12345-1234-1234-1234-abc123456789
```

## Checking Workflow Status

```bash
dapr workflow get \
  --app-id order-processor \
  --workflow-component dapr \
  --workflow-instance-id abc12345-1234-1234-1234-abc123456789
```

Sample output:

```
{
  "instanceID": "abc12345-1234-1234-1234-abc123456789",
  "workflowName": "OrderFulfillmentWorkflow",
  "createdAt": "2026-03-31T10:00:00Z",
  "lastUpdatedAt": "2026-03-31T10:00:05Z",
  "runtimeStatus": "RUNNING"
}
```

## Pausing a Running Workflow

```bash
dapr workflow pause \
  --app-id order-processor \
  --workflow-component dapr \
  --workflow-instance-id abc12345-1234-1234-1234-abc123456789
```

## Resuming a Paused Workflow

```bash
dapr workflow resume \
  --app-id order-processor \
  --workflow-component dapr \
  --workflow-instance-id abc12345-1234-1234-1234-abc123456789
```

## Terminating a Workflow

Force-stop a running or stuck workflow:

```bash
dapr workflow terminate \
  --app-id order-processor \
  --workflow-component dapr \
  --workflow-instance-id abc12345-1234-1234-1234-abc123456789
```

## Purging Workflow History

Remove all history for a completed or terminated workflow:

```bash
dapr workflow purge \
  --app-id order-processor \
  --workflow-component dapr \
  --workflow-instance-id abc12345-1234-1234-1234-abc123456789
```

## Scripted Workflow Automation

Start a workflow and poll for completion:

```bash
#!/bin/bash
INSTANCE_ID=$(dapr workflow run \
  --app-id order-processor \
  --workflow-component dapr \
  --workflow-name OrderFulfillmentWorkflow \
  --input '{"orderId":"ord-999"}' | grep "Instance ID" | awk '{print $NF}')

echo "Started workflow: $INSTANCE_ID"

while true; do
  STATUS=$(dapr workflow get \
    --app-id order-processor \
    --workflow-component dapr \
    --workflow-instance-id $INSTANCE_ID | jq -r '.runtimeStatus')

  echo "Status: $STATUS"
  if [[ "$STATUS" == "COMPLETED" || "$STATUS" == "FAILED" ]]; then
    break
  fi
  sleep 2
done
```

## Summary

The `dapr workflow` CLI command provides complete lifecycle management for workflow instances without requiring application code changes or API calls. It is particularly useful for operational tasks like pausing workflows during maintenance windows, terminating stuck instances, and purging old workflow history to manage storage costs.
