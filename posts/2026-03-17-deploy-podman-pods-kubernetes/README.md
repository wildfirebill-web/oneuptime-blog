# How to Deploy Podman Pods to Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Kubernetes, Deployment, YAML

Description: Learn how to export Podman pods as Kubernetes YAML and deploy them to a Kubernetes cluster.

---

> Podman bridges local development and production by generating Kubernetes-ready YAML from running pods.

Podman lets you develop and test pods locally, then export them as standard Kubernetes YAML for deployment to a real cluster. The `podman generate kube` command produces manifests that work with `kubectl apply`, making the transition from local development to production seamless.

---

## Creating a Pod Locally

```bash
# Create a pod with multiple containers
podman pod create --name webapp -p 8080:80

# Add an nginx frontend
podman run -d --pod webapp --name frontend docker.io/library/nginx:alpine

# Add a Redis backend
podman run -d --pod webapp --name cache docker.io/library/redis:7-alpine

# Verify the pod is running
podman pod ps
```

## Generating Kubernetes YAML

```bash
# Export the running pod as Kubernetes YAML
podman generate kube webapp > webapp-k8s.yaml

# View the generated YAML
cat webapp-k8s.yaml
```

The generated YAML includes the Pod spec with all containers, ports, and volume mounts.

## Pushing Images to a Registry

Before deploying to Kubernetes, push your custom images to a container registry.

```bash
# Tag the local image for a registry
podman tag localhost/myapp:latest registry.example.com/myapp:v1.0

# Push to the registry
podman push registry.example.com/myapp:v1.0

# Update the generated YAML to reference the registry image
sed -i 's|localhost/myapp:latest|registry.example.com/myapp:v1.0|g' webapp-k8s.yaml
```

## Deploying to Kubernetes

```bash
# Apply the generated YAML to your cluster
kubectl apply -f webapp-k8s.yaml

# Verify the pod is running in Kubernetes
kubectl get pods
kubectl describe pod webapp
```

## Generating a Deployment Instead

For production, you typically want a Deployment rather than a bare Pod.

```bash
# Generate the YAML
podman generate kube webapp > base-pod.yaml

# Wrap it in a Deployment manually or use this pattern:
cat > webapp-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: frontend
          image: registry.example.com/frontend:v1.0
          ports:
            - containerPort: 80
        - name: cache
          image: docker.io/library/redis:7-alpine
EOF

kubectl apply -f webapp-deployment.yaml
```

## Adding a Service

```bash
# Create a Service to expose the deployment
cat > webapp-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
EOF

kubectl apply -f webapp-service.yaml
```

## Full Workflow

```bash
# 1. Develop locally with Podman
podman pod create --name myapp -p 8080:80
podman run -d --pod myapp --name web docker.io/library/nginx:alpine

# 2. Test locally
curl http://localhost:8080

# 3. Generate Kubernetes YAML
podman generate kube myapp > myapp-k8s.yaml

# 4. Push images and deploy
kubectl apply -f myapp-k8s.yaml
kubectl get pods
```

## Summary

Use `podman generate kube` to export locally developed pods as Kubernetes YAML. Push custom images to a registry, update image references in the YAML, and deploy with `kubectl apply`. This workflow lets you develop with Podman and deploy to any Kubernetes cluster.
