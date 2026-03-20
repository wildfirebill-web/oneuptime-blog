# How to Set Up Tilt for Local Development with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Tilt, Development, Inner Loop

Description: Configure Tilt for an enhanced local Kubernetes development experience with live updates, multi-service orchestration, and a real-time dashboard for Rancher clusters.

## Introduction

Tilt is a development tool that orchestrates multi-service Kubernetes applications with live updates and a visual dashboard. Unlike Skaffold, Tilt's Tiltfile uses the Starlark language (Python dialect) for full programmability. This guide covers setting up Tilt for development against a Rancher cluster.

## Prerequisites

- Tilt CLI installed
- kubectl configured for your Rancher cluster
- Docker configured (or Kaniko for in-cluster builds)
- A development namespace in Rancher

## Step 1: Install Tilt

```bash
# macOS

brew install tilt

# Linux
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash

# Verify
tilt version
```

## Step 2: Create a Tiltfile

```python
# Tiltfile - Main Tilt configuration (Python-like Starlark)

# Allow building against a remote cluster
allow_k8s_contexts('rancher-development')

# Build the frontend image
docker_build(
    'registry.example.com/frontend',
    './frontend',
    dockerfile='./frontend/Dockerfile.dev',
    # Live update: sync changes without full rebuild
    live_update=[
        sync('./frontend/src', '/app/src'),
        sync('./frontend/public', '/app/public'),
        # Run npm install when package.json changes
        run('cd /app && npm install', trigger=['./frontend/package.json']),
    ],
)

# Build the backend image
docker_build(
    'registry.example.com/backend',
    './backend',
    dockerfile='./backend/Dockerfile',
    live_update=[
        sync('./backend/src', '/app/src'),
        # Restart the process when Python files change
        restart_process_trigger('./backend/src/**/*.py'),
    ],
)

# Deploy using Helm
helm_resource(
    'my-app',
    './chart',
    image_deps=['registry.example.com/frontend', 'registry.example.com/backend'],
    image_keys=[('frontend.image.repository', 'frontend.image.tag'),
                ('backend.image.repository', 'backend.image.tag')],
    flags=[
        '--namespace', 'development',
        '--set', 'frontend.enabled=true',
        '--set', 'backend.enabled=true',
        '--set', 'replicaCount=1',
    ],
)

# Port forwards
k8s_resource('my-app', port_forwards=[
    port_forward(3000, 3000, 'Frontend'),
    port_forward(8080, 8080, 'Backend'),
])

# Add links to the Tilt dashboard
k8s_resource('my-app', links=[
    link('http://localhost:3000', 'Frontend'),
    link('http://localhost:8080/api/docs', 'API Docs'),
])
```

## Step 3: Multi-Service Development

```python
# Tiltfile - Multi-service application
# Load helper extensions
load('ext://namespace', 'namespace_create')
load('ext://helm_resource', 'helm_resource', 'helm_repo')

# Ensure development namespace exists
namespace_create('development')

# Group services
# Frontend
docker_build('registry.example.com/frontend', './frontend',
    live_update=[sync('./frontend/src', '/app/src')],
    cache_from=['registry.example.com/frontend:cache'],
)

# Backend API
docker_build('registry.example.com/api', './api',
    live_update=[
        sync('./api/src', '/app'),
        run('pip install -r /app/requirements.txt',
            trigger=['./api/requirements.txt']),
    ],
)

# Background worker
docker_build('registry.example.com/worker', './worker',
    live_update=[sync('./worker/src', '/app')],
)

# Apply Kubernetes manifests
k8s_yaml(['./k8s/frontend.yaml',
           './k8s/api.yaml',
           './k8s/worker.yaml'])

# Configure each service
k8s_resource('frontend',
    port_forwards='3000:3000',
    resource_deps=['api'],  # Start api first
    labels=['frontend'],
)

k8s_resource('api',
    port_forwards='8080:8080',
    resource_deps=['redis', 'database'],
    labels=['backend'],
)

k8s_resource('worker',
    resource_deps=['redis', 'kafka'],
    labels=['backend'],
)

# External dependencies (databases, etc.)
# Watch them but don't manage them
k8s_resource('redis', labels=['infrastructure'])
k8s_resource('postgresql', labels=['infrastructure'])
```

## Step 4: Configure Live Updates

```python
# Tiltfile - Advanced live update configuration

def backend_build(name, path):
    """Helper function for Python backend builds"""
    docker_build(
        'registry.example.com/' + name,
        path,
        dockerfile=path + '/Dockerfile',
        live_update=[
            # Only sync Python files
            sync(path + '/src', '/app/src'),
            # Restart uWSGI when source changes
            run('kill -HUP 1'),  # Send SIGHUP to reload
        ],
        ignore=['**/__pycache__', '**/*.pyc', '**/tests/'],
    )

backend_build('order-service', './services/orders')
backend_build('payment-service', './services/payments')
backend_build('notification-service', './services/notifications')
```

## Step 5: Custom Triggers and CI-like Tests

```python
# Tiltfile - Run tests as part of the development loop

# Run tests when source changes
local_resource('unit-tests',
    'pytest services/api/tests/unit/ -v',
    deps=['./services/api/src', './services/api/tests/unit'],
    labels=['tests'],
)

# Run integration tests less frequently
local_resource('integration-tests',
    'pytest services/api/tests/integration/ -v',
    deps=['./services/api/src', './services/api/tests/integration'],
    trigger_mode=TRIGGER_MODE_MANUAL,  # Only run when manually triggered
    labels=['tests'],
)

# Linting
local_resource('lint',
    'flake8 services/ && black --check services/',
    deps=['./services'],
    labels=['quality'],
)
```

## Step 6: Manage Secrets in Development

```python
# Tiltfile - Development secrets management

# Load secrets from .env files (never commit these!)
def load_dotenv(path):
    f = read_file(path, default='')
    for line in f.split('\n'):
        line = line.strip()
        if line and not line.startswith('#') and '=' in line:
            key, value = line.split('=', 1)
            os.putenv(key, value)

load_dotenv('.env.development')

# Create Kubernetes secret from environment
k8s_yaml(
    local('kubectl create secret generic dev-secrets \
        --from-literal=db_password=$DB_PASSWORD \
        --from-literal=api_key=$API_KEY \
        -n development \
        --dry-run=client -o yaml'),
)
```

## Step 7: Run and Access the Dashboard

```bash
# Start Tilt development loop
tilt up

# Access the Tilt dashboard
# Opens automatically at: http://localhost:10350

# View logs for a specific service
tilt logs api

# Trigger a manual resource
tilt trigger integration-tests

# Stop Tilt
tilt down  # Tears down all Tilt-managed resources
```

## Troubleshooting

```bash
# Check Tilt logs
tilt log

# Verify cluster context
tilt get cluster

# Check if Tilt can access the registry
tilt docker-prune

# Debug a failing build
tilt trigger <resource-name>
```

## Conclusion

Tilt provides a comprehensive development experience for Kubernetes applications with its visual dashboard, live updates, and programmable Tiltfile configuration. The Starlark-based configuration enables powerful abstractions and code reuse across projects. For teams developing on Rancher clusters, Tilt's multi-service orchestration and test integration capabilities significantly reduce context switching and speed up the development feedback loop.
