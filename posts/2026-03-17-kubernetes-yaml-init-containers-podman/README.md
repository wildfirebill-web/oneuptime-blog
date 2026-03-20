# How to Use Kubernetes YAML with Init Containers in Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Kubernetes, Init Containers, YAML

Description: Learn how to define and run init containers in Kubernetes YAML using podman kube play for pre-start initialization tasks.

---

> Init containers run to completion before the main application containers start, making them ideal for setup tasks like migrations and config generation.

Init containers are specialized containers that run before application containers in a pod. They are perfect for tasks like waiting for a dependency, running database migrations, or generating configuration files. Podman supports init containers in Kubernetes YAML through `podman kube play`.

---

## Basic Init Container

```yaml
# pod-with-init.yaml

apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
    - name: init-permissions
      image: docker.io/library/busybox:latest
      command: ["sh", "-c", "mkdir -p /data/cache && chmod 777 /data/cache"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
  containers:
    - name: app
      image: docker.io/library/nginx:alpine
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: shared-data
      emptyDir: {}
```

```bash
# Deploy the pod - init container runs first
podman kube play pod-with-init.yaml

# Check that the init container ran and completed
podman ps -a --filter name=myapp-init
```

## Waiting for a Dependency

```yaml
# pod-wait-for-db.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  initContainers:
    - name: wait-for-db
      image: docker.io/library/busybox:latest
      command:
        - "sh"
        - "-c"
        - |
          echo "Waiting for database to be ready..."
          until nc -z db-host 5432; do
            echo "DB not ready, retrying in 2s..."
            sleep 2
          done
          echo "Database is ready!"
  containers:
    - name: app
      image: docker.io/library/python:3.12-slim
      command: ["python", "-m", "http.server", "8000"]
```

## Multiple Init Containers

Init containers run sequentially in the order they are defined.

```yaml
# pod-multi-init.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fullstack
spec:
  initContainers:
    - name: init-config
      image: docker.io/library/busybox:latest
      command: ["sh", "-c", "echo 'port=8080' > /config/app.conf"]
      volumeMounts:
        - name: config
          mountPath: /config
    - name: init-data
      image: docker.io/library/busybox:latest
      command: ["sh", "-c", "echo 'Welcome' > /www/index.html"]
      volumeMounts:
        - name: webroot
          mountPath: /www
  containers:
    - name: web
      image: docker.io/library/nginx:alpine
      volumeMounts:
        - name: config
          mountPath: /etc/app
        - name: webroot
          mountPath: /usr/share/nginx/html
  volumes:
    - name: config
      emptyDir: {}
    - name: webroot
      emptyDir: {}
```

```bash
# Deploy and verify init containers ran in order
podman kube play pod-multi-init.yaml

# Check the generated files
podman exec fullstack-web cat /etc/app/app.conf
# Output: port=8080

podman exec fullstack-web cat /usr/share/nginx/html/index.html
# Output: Welcome
```

## Verifying Init Container Logs

```bash
# View logs from an init container
podman logs fullstack-init-config

# View logs from the second init container
podman logs fullstack-init-data
```

## Summary

Podman supports Kubernetes init containers through `podman kube play`. Init containers run sequentially before application containers, making them useful for setup tasks like configuration generation, permission setting, and waiting for dependencies to become available.
