# How to Configure Hot Reloading for Applications on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Hot Reload, Development, Live Update

Description: Configure hot reloading for Node.js, Python, Go, and Java applications running in Rancher Kubernetes clusters to speed up your development feedback loop.

## Introduction

Hot reloading allows your application to reflect code changes instantly without restarting the container or rebuilding the Docker image. This guide covers setting up hot reload for various language runtimes in development containers deployed to Rancher, significantly reducing the time between making a change and seeing it in action.

## Prerequisites

- Rancher-managed development cluster
- Applications deployed with development Dockerfiles
- File sync tool (kubectl cp, Skaffold, Telepresence, or DevSpace)

## Step 1: Node.js Hot Reload with Nodemon

```dockerfile
# Dockerfile.dev - Node.js development container
FROM node:18-alpine
WORKDIR /app

# Install nodemon globally
RUN npm install -g nodemon

COPY package*.json ./
RUN npm install

COPY . .

# Use nodemon for hot reload
CMD ["nodemon", "--watch", "src", "--ext", "js,json,ts", "src/index.js"]
```

```yaml
# deployment-dev.yaml - Development deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app-dev
  namespace: development
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          image: registry.example.com/node-app:dev
          volumeMounts:
            - name: source
              mountPath: /app/src
          env:
            - name: NODE_ENV
              value: development
      volumes:
        - name: source
          emptyDir: {}
```

## Step 2: Python Hot Reload with Uvicorn

```dockerfile
# Dockerfile.dev - Python FastAPI development container
FROM python:3.11-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt uvicorn[standard]

COPY . .

# --reload flag enables hot reload
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload", "--reload-dir", "/app"]
```

```yaml
# Deployment with source volume for sync
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-api-dev
  namespace: development
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: api
          image: registry.example.com/python-api:dev
          ports:
            - containerPort: 8000
          env:
            - name: PYTHONPATH
              value: /app
            - name: LOG_LEVEL
              value: debug
```

## Step 3: Go Hot Reload with Air

```dockerfile
# Dockerfile.dev - Go development container with Air
FROM golang:1.21-alpine
WORKDIR /app

# Install Air for hot reloading
RUN go install github.com/cosmtrek/air@latest

COPY go.mod go.sum ./
RUN go mod download

COPY . .

CMD ["air", "-c", ".air.toml"]
```

```toml
# .air.toml - Air configuration
root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
cmd = "go build -o ./tmp/main ."
bin = "tmp/main"
full_bin = "APP_ENV=dev APP_USER=air ./tmp/main"
include_ext = ["go", "tpl", "tmpl", "html"]
exclude_dir = ["assets", "tmp", "vendor", "testdata"]
include_dir = []
exclude_file = []
exclude_regex = ["_test.go"]
exclude_unchanged = false
follow_symlink = false
log = "build-errors.log"
delay = 0
stop_on_error = false
send_interrupt = false
kill_delay = "0s"
rerun = false
rerun_delay = 500

[log]
time = false
main_only = false

[color]
main = "magenta"
watcher = "cyan"
build = "yellow"
runner = "green"

[misc]
clean_on_exit = false
poll = false
poll_interval = 0
```

## Step 4: Java Hot Reload with Spring DevTools

```dockerfile
# Dockerfile.dev - Spring Boot development container
FROM eclipse-temurin:17-jdk
WORKDIR /app

# Enable Spring DevTools remote
COPY target/*.jar app.jar

# Spring DevTools enables automatic restart
CMD ["java", \
     "-Dspring.profiles.active=dev", \
     "-Dspring.devtools.restart.enabled=true", \
     "-Dspring.devtools.livereload.enabled=true", \
     "-jar", "app.jar"]
```

```yaml
# application-dev.yml - Spring Boot dev config
spring:
  devtools:
    restart:
      enabled: true
      poll-interval: 1000ms
      quiet-period: 400ms
    livereload:
      enabled: true
  jpa:
    show-sql: true
```

## Step 5: Sync Files with kubectl

```bash
# Sync files directly to the running container
kubectl cp ./src/ development/$(kubectl get pod -n development -l app=node-app -o jsonpath='{.items[0].metadata.name}'):/app/src/

# Watch and sync with inotifywait (Linux) or fswatch (macOS)
# macOS
fswatch -o ./src | xargs -n1 -I{} kubectl cp ./src/ \
  development/$(kubectl get pod -n development -l app=node-app \
  -o jsonpath='{.items[0].metadata.name}'):/app/src/
```

## Step 6: Using Skaffold File Sync

```yaml
# skaffold.yaml - Configure file sync for hot reload
apiVersion: skaffold/v4beta11
kind: Config
build:
  artifacts:
    - image: registry.example.com/node-app
      docker:
        dockerfile: Dockerfile.dev
      sync:
        manual:
          - src: "src/**/*.js"
            dest: /app/src
          - src: "src/**/*.ts"
            dest: /app/src
```

## Step 7: Configure Resource Requests for Dev

```yaml
# Ensure sufficient resources for dev tooling
containers:
  - name: app
    image: registry.example.com/app:dev
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi
    readinessProbe:
      # Longer timeout for hot reload restarts
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 10
```

## Conclusion

Hot reloading dramatically speeds up the development feedback loop for Kubernetes applications on Rancher. By choosing the appropriate hot reload solution for each language runtime—Nodemon for Node.js, Uvicorn's reload flag for Python, Air for Go, and Spring DevTools for Java—you can achieve near-instant code updates. Combined with file sync tools like Skaffold or Telepresence, this workflow provides a local development experience while running on the actual Rancher cluster infrastructure.
