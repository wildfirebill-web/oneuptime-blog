# How to Configure Hot Reloading for Applications on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Hot Reloading, Developer Experience, Kubernetes, File Sync, Skaffold

Description: Configure hot reloading for Node.js, Python, and Go applications deployed on Rancher using file sync tools and process managers for near-instant code iteration.

## Introduction

Hot reloading allows code changes to take effect in a running application without restarting the process or rebuilding the container. On Kubernetes, this requires file sync to propagate local changes into the running container, combined with a process manager that watches for file changes.

## Approach 1: Skaffold Live Update (Recommended)

Skaffold's `live_update` feature syncs file changes directly into the running container:

```yaml
# skaffold.yaml
build:
  artifacts:
    - image: myregistry/nodeapp
      docker:
        dockerfile: Dockerfile.dev    # Dev Dockerfile with nodemon
      sync:
        manual:
          - src: 'src/**/*.js'       # Sync JS changes without rebuilding
            dest: /app/src
          - src: 'src/**/*.ts'
            dest: /app/src
```

### Development Dockerfile with Nodemon

```dockerfile
# Dockerfile.dev
FROM node:20-alpine

WORKDIR /app

# Install nodemon for hot reloading
RUN npm install -g nodemon

COPY package*.json ./
RUN npm install

COPY . .

# Use nodemon to watch for file changes
CMD ["nodemon", "--watch", "src", "--exec", "node", "src/index.js"]
```

## Approach 2: Python Hot Reload with Uvicorn

```dockerfile
# Dockerfile.dev for Python/FastAPI
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

# Uvicorn with --reload flag watches for changes
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

```yaml
# skaffold.yaml for Python
build:
  artifacts:
    - image: myregistry/pyapp
      sync:
        manual:
          - src: '**/*.py'    # Sync Python files directly
            dest: /app
```

## Approach 3: Tilt with Live Update

```python
# Tiltfile
docker_build(
    'myregistry/goapp',
    '.',
    live_update=[
        # Sync Go source files
        sync('./cmd', '/app/cmd'),
        sync('./internal', '/app/internal'),
        # Re-run go build and restart when binary changes
        run('go build -o /app/server ./cmd/server && pkill server || true; /app/server &'),
    ]
)
```

## Approach 4: Telepresence for File Sync

```bash
# Connect to cluster
telepresence connect

# Intercept the service
telepresence intercept myapp --port 8080

# Run locally with your IDE's hot reload
npm run dev    # Changes immediately affect the intercepted traffic
```

## Development vs Production Images

Always use separate Dockerfiles for development and production:

```yaml
# skaffold.yaml - profile-based image selection
profiles:
  - name: dev
    build:
      artifacts:
        - image: myregistry/myapp
          docker:
            dockerfile: Dockerfile.dev    # With hot reload tools
  - name: prod
    build:
      artifacts:
        - image: myregistry/myapp
          docker:
            dockerfile: Dockerfile        # Optimized production image
```

## Conclusion

Hot reloading on Rancher requires two things working together: a file sync tool (Skaffold, Tilt, or Telepresence) to move changes into the cluster, and a process watcher (nodemon, uvicorn --reload, air for Go) to reload the application when files change. This combination provides iteration speeds comparable to local development.
