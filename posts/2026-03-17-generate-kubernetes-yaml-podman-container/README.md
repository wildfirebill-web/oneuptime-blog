# How to Generate Kubernetes YAML from a Podman Container

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Kubernetes, YAML, Migration

Description: Learn how to generate Kubernetes-compatible YAML manifests from running Podman containers.

---

> Podman can export a running container's configuration as Kubernetes YAML, bridging the gap between local development and cluster deployment.

One of Podman's standout features is its ability to generate Kubernetes YAML from running containers. This lets you develop and test locally with Podman, then export the configuration directly to Kubernetes manifests. The `podman generate kube` command handles the translation.

---

## Generating YAML from a Running Container

```bash
# Run a container with some configuration

podman run -d --name my-web \
  -p 8080:80 \
  -e APP_ENV=production \
  -v web-data:/usr/share/nginx/html \
  docker.io/library/nginx:alpine

# Generate Kubernetes YAML
podman generate kube my-web
```

## Saving the YAML to a File

```bash
# Save the generated YAML to a file
podman generate kube my-web > my-web-pod.yaml

# View the generated file
cat my-web-pod.yaml
```

## Understanding the Generated Output

The generated YAML includes a Pod spec with the container's image, ports, environment variables, and volume mounts.

```yaml
# Example output structure:
# apiVersion: v1
# kind: Pod
# metadata:
#   name: my-web
# spec:
#   containers:
#     - name: my-web
#       image: docker.io/library/nginx:alpine
#       ports:
#         - containerPort: 80
#           hostPort: 8080
#       env:
#         - name: APP_ENV
#           value: production
#       volumeMounts:
#         - mountPath: /usr/share/nginx/html
#           name: web-data
```

## Generating YAML with Service Definition

```bash
# Include a Kubernetes Service definition in the output
podman generate kube --service my-web > my-web-with-service.yaml

# The output includes both a Pod and a Service resource
```

## Generating YAML from a Container with Multiple Ports

```bash
# Run a container with multiple port mappings
podman run -d --name multi-port \
  -p 8080:80 \
  -p 8443:443 \
  docker.io/library/nginx:alpine

# Generate YAML - all ports are included
podman generate kube multi-port
```

## Generating YAML from a Container with Environment Variables

```bash
# Run a container with several environment variables
podman run -d --name api-server \
  -e DATABASE_HOST=db.example.com \
  -e DATABASE_PORT=5432 \
  -e LOG_LEVEL=info \
  -e APP_PORT=3000 \
  docker.io/library/node:20-alpine sleep 3600

# Generate YAML - all env vars are captured
podman generate kube api-server > api-server.yaml
```

## Applying the Generated YAML to Kubernetes

```bash
# Generate the YAML
podman generate kube my-web > deployment.yaml

# Apply it to a Kubernetes cluster
kubectl apply -f deployment.yaml

# Or replay it locally with Podman
podman play kube deployment.yaml
```

## Summary

Use `podman generate kube` to export a running container's configuration as Kubernetes-compatible YAML. Add `--service` to include a Service definition. Save the output to a file and apply it to Kubernetes with `kubectl apply` or replay it with `podman play kube`. This bridges local development and production deployment.
