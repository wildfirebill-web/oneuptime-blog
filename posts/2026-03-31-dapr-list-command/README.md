# How to Use the dapr list Command to View Running Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CLI, Monitoring, Development, Process Management

Description: Learn how to use the dapr list command to view all running Dapr applications and their sidecar details in self-hosted and Kubernetes environments.

---

## Overview

The `dapr list` command shows all currently running Dapr applications along with their app IDs, HTTP and gRPC ports, process IDs, and uptime. It is an essential diagnostic tool for verifying that your services started correctly during local development.

## Basic Usage

List all running Dapr applications in self-hosted mode:

```bash
dapr list
```

Sample output:

```text
  APP ID           HTTP PORT  GRPC PORT  APP PORT  COMMAND        AGE  PID
  order-service    3500       50001      8080      node index.js  10s  12345
  inventory-svc    3501       50002      8081      python app.py  10s  12346
  frontend         3502       50003      3000      npm start      10s  12347
```

## Listing Apps in Kubernetes

```bash
dapr list --kubernetes
```

Or the shorthand:

```bash
dapr list -k
```

Sample output:

```text
  NAMESPACE  APP ID           HTTP PORT  GRPC PORT  AGE  CREATED
  default    order-service    80         50001      2h   2026-03-31 10:00:00
  default    inventory-svc    80         50001      2h   2026-03-31 10:00:01
```

## Listing Apps in a Specific Namespace

```bash
dapr list --kubernetes --namespace production
```

## JSON Output for Scripting

Output the list as JSON for use in scripts or monitoring pipelines:

```bash
dapr list --output json
```

Sample JSON output:

```json
[
  {
    "appId": "order-service",
    "httpPort": 3500,
    "grpcPort": 50001,
    "appPort": 8080,
    "command": "node index.js",
    "age": "15s",
    "pid": 12345,
    "created": "2026-03-31 10:00:00"
  }
]
```

## Using dapr list in Health Check Scripts

```bash
#!/bin/bash
REQUIRED_APPS=("order-service" "inventory-service")

RUNNING=$(dapr list --output json | jq -r '.[].appId')

for app in "${REQUIRED_APPS[@]}"; do
  if echo "$RUNNING" | grep -q "$app"; then
    echo "$app is running"
  else
    echo "ERROR: $app is NOT running"
    exit 1
  fi
done
```

## Checking a Specific App

There is no direct filter flag, but you can use grep:

```bash
dapr list | grep order-service
```

## Wide Output

For more detail in the terminal:

```bash
dapr list --output wide
```

This adds additional columns such as the process command and component count.

## Summary

`dapr list` is your first debugging step when services are not behaving as expected. It shows which apps are running, their port assignments, and PIDs. Combined with JSON output and scripting, it can serve as the basis for automated health checks in development pipelines.
