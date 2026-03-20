# How to Deploy Edge Stacks to Multiple Environments in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Edge Computing, DevOps, Stacks

Description: Learn how to use Portainer Edge Stacks to deploy containerized applications to multiple remote edge environments simultaneously.

## Introduction

Portainer Edge Stacks allow you to define a Docker Compose application and deploy it across many edge endpoints at once — without SSH-ing into each device. This is invaluable for managing retail stores, factory floors, or any distributed environment where dozens or hundreds of devices need identical (or parameterized) workloads.

## Prerequisites

- Portainer Business Edition with Edge Compute enabled
- Multiple Portainer Edge Agents connected
- Edge Groups configured (see the Edge Groups guide)

## Understanding Edge Stack Deployment

When you deploy an Edge Stack:
1. The Portainer server stores the stack definition.
2. Each edge agent polls the server (or receives a push) and downloads the stack.
3. Docker Compose runs the stack locally on the edge device.
4. Status is reported back to Portainer.

This model works even for devices with intermittent connectivity.

## Step 1: Prepare Your Stack File

Create a `docker-compose.yml` for the application you want to deploy. Use environment variables for values that differ between environments.

```yaml
# docker-compose.yml for edge deployment
# Uses environment variables for per-device customization
version: "3.8"

services:
  app:
    image: myorg/myapp:${APP_VERSION:-latest}
    restart: always
    ports:
      - "8080:8080"
    environment:
      - DEVICE_ID=${DEVICE_ID}
      - SITE_NAME=${SITE_NAME}
      - LOG_LEVEL=${LOG_LEVEL:-info}
    volumes:
      - app_data:/data

  prometheus-exporter:
    image: prom/node-exporter:latest
    restart: always
    ports:
      - "9100:9100"

volumes:
  app_data:
```

## Step 2: Create an Edge Stack

1. Navigate to **Edge Compute > Edge Stacks**.
2. Click **Add edge stack**.
3. Enter a **Name** for the stack (e.g., `retail-pos-app`).
4. Choose the **build method**:
   - **Web editor** — paste your compose YAML directly.
   - **Upload** — upload a compose file from your machine.
   - **Repository** — pull from a Git repo (recommended for CI/CD).

## Step 3: Select Target Edge Groups

```
# In the Portainer UI, after entering the stack name and content:
# - Select one or more Edge Groups under "Edge groups"
# - Example: select "US-Retail-Stores" and "EU-Retail-Stores"
# This deploys the stack to ALL endpoints in both groups
```

5. Under **Edge groups**, select the groups that represent your target environments (e.g., `Production-EU`, `Production-US`).
6. Optionally set **Pre-pull images** to ensure images are pulled before starting.

## Step 4: Configure Per-Environment Variables

If you use different variable values per environment, you can set **Environment variables** at the edge group or endpoint level:

```bash
# Example: set per-device env vars when deploying the edge agent
docker run -d \
  --name portainer_edge_agent \
  -e EDGE_KEY="device_edge_key" \
  -e DEVICE_ID="store-berlin-001" \
  -e SITE_NAME="Berlin Main Store" \
  -e APP_VERSION="2.4.1" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/agent:latest
```

In Portainer, you can also override stack-level environment variables per endpoint.

## Step 5: Monitor Deployment Status

After creating the stack:
1. Go to **Edge Compute > Edge Stacks**.
2. Click your stack name.
3. View the deployment status per endpoint:
   - **OK** — Stack is running.
   - **Deploying** — In progress.
   - **Failed** — Check the edge device logs.

## Step 6: Update the Stack Across All Environments

To roll out a new version:
1. Edit the stack (change image tag or compose content).
2. Click **Update the stack**.
3. Portainer propagates the update to all assigned edge groups.

```yaml
# Updated stack — bump APP_VERSION for a rolling update
services:
  app:
    image: myorg/myapp:2.5.0  # Updated from 2.4.1
    restart: always
```

## Best Practices

- **Pin image versions** in edge stacks to avoid unexpected updates from `:latest` tags.
- **Use Git repositories** as the stack source for version-controlled rollouts.
- **Test in a staging edge group** before promoting to production edge groups.
- **Monitor failed deployments** promptly — edge devices may have resource constraints.

## Conclusion

Portainer Edge Stacks drastically simplify multi-environment deployments. Instead of coordinating SSH sessions or custom scripts, you define your application once and target the right edge groups. The result is consistent, auditable, and easily updatable deployments across your entire device fleet.
