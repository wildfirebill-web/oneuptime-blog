# How to Auto-Build Images with podman kube play

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Kubernetes, Build, Images

Description: Learn how to automatically build container images from a Containerfile when deploying with podman kube play using the --build flag.

---

> The --build flag tells podman kube play to build images from local Containerfiles before deploying your pods.

During development you often need to rebuild images and redeploy pods in a tight loop. Instead of running separate build and play commands, `podman kube play --build` combines both steps. Podman looks for a build context matching the image name and builds it on the fly.

---

## Project Structure

```text
my-app/
├── Containerfile
├── app.py
└── kube.yaml
```

## The Containerfile

```dockerfile
# Containerfile

FROM docker.io/library/python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

## The Kubernetes YAML

```yaml
# kube.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: app
      # Use a local image name (not from a registry)
      image: localhost/myapp:latest
      ports:
        - containerPort: 8080
```

## Building and Deploying

```bash
# Build the image and deploy the pod in one command
# The --build flag triggers a build for local images
podman kube play --build kube.yaml

# Podman looks for a directory named "myapp" with a Containerfile
# or uses the current directory if the image matches
```

## Specifying a Build Context

```bash
# Use --context-dir to point to the build context directory
podman kube play --build --context-dir ./my-app kube.yaml

# Podman builds localhost/myapp:latest from ./my-app/Containerfile
# then deploys the pod using the freshly built image
```

## Rebuild and Redeploy Cycle

```bash
# Tear down the existing deployment
podman kube play --down kube.yaml

# Rebuild and redeploy after code changes
podman kube play --build kube.yaml
```

## Using Annotations for Build Context

```yaml
# kube-annotated.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    # Tell Podman where to find the Containerfile
    io.podman.annotations.build.context.dir: "./my-app"
spec:
  containers:
    - name: app
      image: localhost/myapp:latest
```

## Multiple Containers with Builds

```yaml
# multi-build.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fullstack
spec:
  containers:
    - name: frontend
      image: localhost/frontend:latest
    - name: backend
      image: localhost/backend:latest
```

```bash
# Build both images and deploy
# Podman looks for frontend/ and backend/ directories
podman kube play --build multi-build.yaml
```

## Summary

The `--build` flag on `podman kube play` builds container images from local Containerfiles before deploying pods. Use `--context-dir` to specify the build context directory. This eliminates the need for separate build and deploy steps during development.
