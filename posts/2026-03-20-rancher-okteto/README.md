# How to Use Okteto with Rancher for Remote Development - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Okteto, Development, Remote Development

Description: Use Okteto with Rancher to develop applications directly in Kubernetes with instant hot reloading, bypassing the container build process entirely.

## Introduction

Okteto replaces your running container with a development container that has your source code and tools mounted. Changes to your local code are instantly reflected in the remote container without rebuilding images. This guide covers integrating Okteto with Rancher-managed clusters for a seamless remote development experience.

## Prerequisites

- Okteto CLI installed
- kubectl configured for your Rancher cluster
- A running application deployed to Rancher
- Source code for the application

## Step 1: Install Okteto CLI

```bash
# macOS

brew install okteto

# Linux
curl https://get.okteto.com -sSfL | sh

# Windows (via scoop)
scoop install okteto

# Verify
okteto version
```

## Step 2: Configure Okteto Context

```bash
# Set Okteto context to use your Rancher cluster
okteto context use https://rancher.example.com --token <api-token>

# Or use existing kubeconfig
okteto context use --context rancher-production

# List available contexts
okteto context list

# Show current context
okteto context show
```

## Step 3: Create okteto.yml

```yaml
# okteto.yml - Okteto development configuration
dev:
  backend:
    # Replace this image with a dev container
    image: registry.example.com/backend:dev-tools
    command: bash
    # Sync local source to container
    sync:
      - .:/app
    # Forward ports
    forward:
      - 8080:8080
      - 5678:5678  # Debugger
    # Environment variables
    environment:
      - DEBUG=true
      - LOG_LEVEL=debug
    # Mount local SSH keys (for git operations)
    volumes:
      - /root/.ssh:/root/.ssh:ro
    # Resource limits for dev container
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "2"
        memory: "2Gi"
    # Kubernetes context settings
    context: rancher-development
    namespace: development
```

## Step 4: Start Development Mode

```bash
# Activate development mode for a deployment
okteto up backend

# This will:
# 1. Replace the backend container with a dev container
# 2. Sync your local files to the container
# 3. Open a shell in the container

# Inside the container, start your application
python -m uvicorn main:app --reload --host 0.0.0.0 --port 8080
```

## Step 5: Multi-Service Development

```yaml
# okteto.yml - Multi-service configuration
dev:
  api:
    image: python:3.11-dev
    command: ["python", "-m", "uvicorn", "main:app", "--reload"]
    sync:
      - ./api:/app
    forward:
      - 8000:8000
    workdir: /app
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgresql:5432/mydb
      - REDIS_URL=redis://redis:6379

  worker:
    image: python:3.11-dev
    command: ["celery", "-A", "tasks", "worker", "--loglevel=info"]
    sync:
      - ./worker:/app
    workdir: /app
    environment:
      - BROKER_URL=amqp://guest:guest@rabbitmq:5672//

  frontend:
    image: node:18-alpine
    command: ["npm", "run", "dev"]
    sync:
      - ./frontend/src:/app/src
      - ./frontend/public:/app/public
    forward:
      - 3000:3000
    workdir: /app
```

## Step 6: Configure Persistent Volumes

```yaml
# okteto.yml - Persistent storage for dev dependencies
dev:
  backend:
    image: python:3.11
    command: bash
    sync:
      - .:/app
    # Persist node_modules / pip packages between sessions
    volumes:
      - /app/.venv  # Persist virtual environment
      - /root/.cache/pip  # Persist pip cache
    workdir: /app
```

## Step 7: Work with Existing Deployments

```bash
# List deployable resources
kubectl get deployments -n development

# Activate development on existing deployment
okteto up --name backend --namespace development

# Activate with a specific manifest
okteto up -f ./okteto.yml api

# Deactivate and restore original deployment
okteto down

# Deactivate without restoring (keep dev container)
okteto down --keep-volumes
```

## Step 8: Remote Debugging

```yaml
# okteto-debug.yml - Configure for VS Code remote debugging
dev:
  backend:
    image: python:3.11
    command: ["python", "-m", "debugpy", "--listen", "0.0.0.0:5678", "-m", "uvicorn", "main:app"]
    sync:
      - .:/app
    forward:
      - 5678:5678  # Attach VS Code debugger here
    environment:
      - PYTHONPATH=/app
    workdir: /app
```

```json
// .vscode/launch.json - VS Code debug configuration
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Okteto Remote Attach",
      "type": "python",
      "request": "attach",
      "connect": {
        "host": "localhost",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}",
          "remoteRoot": "/app"
        }
      ]
    }
  ]
}
```

## Step 9: Cleanup

```bash
# Stop development mode and restore original deployment
okteto down

# Remove dev environment completely
okteto destroy

# View Okteto status
okteto status

# View development container logs
okteto logs backend
```

## Conclusion

Okteto transforms the Kubernetes development experience by eliminating the container image rebuild cycle. By running your development tools directly in the cluster alongside production-equivalent services, you get a realistic environment without the operational overhead of maintaining local Kubernetes clusters. The tight integration with existing Kubernetes deployments makes Okteto particularly well-suited for teams already operating on Rancher who want to improve developer productivity without changing their production infrastructure.
