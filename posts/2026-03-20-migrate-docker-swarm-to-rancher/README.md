# How to Migrate from Docker Swarm to Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Docker Swarm, Kubernetes, Migration, Container Orchestration

Description: Learn how to migrate Docker Swarm services and stacks to Rancher-managed Kubernetes clusters, translating Swarm concepts to their Kubernetes equivalents.

## Introduction

Docker Swarm provides basic container orchestration, but Rancher with Kubernetes offers significantly more capabilities: advanced scheduling, richer networking, and a thriving ecosystem of tools. This guide walks you through migrating Swarm services to Kubernetes deployments managed by Rancher.

## Swarm vs. Kubernetes Concept Mapping

| Docker Swarm | Kubernetes (Rancher) |
|---|---|
| Stack | Namespace + Deployments |
| Service | Deployment + Service |
| Task | Pod |
| Secret | Secret |
| Config | ConfigMap |
| Overlay network | NetworkPolicy |
| Ingress routing mesh | Ingress + LoadBalancer |

## Step 1: Export Swarm Stack Configuration

```bash
docker stack ls
docker stack services myapp
docker service inspect myapp_web --pretty
```

Note the image, replica counts, environment variables, volume mounts, and published ports.

## Step 2: Convert Swarm Compose to Kubernetes Manifests

Given a Swarm stack:

```yaml
version: "3.8"

services:
  api:
    image: myapi:2.0.0
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    secrets:
      - db_password
    ports:
      - "8080:8080"
    networks:
      - backend

secrets:
  db_password:
    external: true

networks:
  backend:
    driver: overlay
```

The equivalent Kubernetes Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myapi:2.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: db_password
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
```

## Step 3: Migrate Secrets

In Swarm, secrets are mounted as files. In Kubernetes, use Secrets:

```bash
# Get the secret value from Swarm (if accessible)

docker secret inspect db_password

# Create the Kubernetes Secret
kubectl create secret generic app-secrets \
  --from-literal=db_password="$(cat swarm-secret.txt)" \
  -n myapp
```

In Rancher, navigate to **Secrets** > **Add Secret** to create it through the UI.

## Step 4: Migrate Configs to ConfigMaps

Swarm configs become Kubernetes ConfigMaps:

```bash
kubectl create configmap nginx-config \
  --from-file=nginx.conf=./swarm-config/nginx.conf \
  -n myapp
```

```yaml
volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
volumeMounts:
  - name: nginx-config
    mountPath: /etc/nginx/nginx.conf
    subPath: nginx.conf
```

## Step 5: Create Services and Ingress

Replace Swarm's routing mesh with a Kubernetes Service and Ingress:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: myapp
spec:
  selector:
    app: api
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: myapp
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 8080
```

## Step 6: Deploy via Rancher

1. In Rancher, navigate to your cluster > **Import YAML**.
2. Apply the namespace first, then secrets, then deployments and services.
3. Monitor the rollout in **Workloads** > **Deployments**.

## Step 7: Decommission Swarm

After verifying the Kubernetes deployment:

```bash
# Drain Swarm nodes
docker node update --availability drain <node-id>

# Remove the Swarm stack
docker stack rm myapp
```

## Best Practices

- Migrate one service at a time, validating each before moving to the next.
- Use Rancher's built-in monitoring to compare performance metrics during cutover.
- Keep DNS TTLs low during migration to enable quick rollback.
- Use Rancher namespaces to mirror Swarm stack separation.
- Test in a non-production Kubernetes cluster before the production migration.

## Conclusion

Migrating from Docker Swarm to Rancher-managed Kubernetes brings your workloads to a production-grade orchestration platform. While the concepts map closely, take time to add proper health probes, resource limits, and Kubernetes-native ingress configuration for a robust, long-term deployment.
