# How to Deploy Dapr Rust Applications on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Rust, Kubernetes, Deployment, Container

Description: Learn how to build a minimal Docker image for a Dapr Rust application and deploy it to Kubernetes with sidecar injection and component configuration.

---

## Introduction

Rust applications produce small, self-contained binaries that make ideal container images. Combining this with Dapr on Kubernetes gives you a high-performance microservice with minimal resource footprint. This guide walks through multi-stage Docker builds, Kubernetes manifests, and Dapr sidecar configuration.

## Multi-Stage Dockerfile

```dockerfile
# Build stage
FROM rust:1.78-alpine AS builder

RUN apk add --no-cache musl-dev
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src

RUN cargo build --release --target x86_64-unknown-linux-musl

# Runtime stage - minimal image
FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/order-service /order-service
EXPOSE 8080
ENTRYPOINT ["/order-service"]
```

```bash
docker build -t my-registry/order-service-rust:v1.0 .
docker push my-registry/order-service-rust:v1.0
```

## Dapr Component Manifests

```yaml
# k8s/components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis:6379
    - name: actorStateStore
      value: "true"
```

```yaml
# k8s/components/pubsub.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: default
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis:6379
```

## Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-rust
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service-rust
  template:
    metadata:
      labels:
        app: order-service-rust
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
        dapr.io/log-level: "warn"
        dapr.io/sidecar-cpu-request: "50m"
        dapr.io/sidecar-cpu-limit: "200m"
        dapr.io/sidecar-memory-request: "64Mi"
        dapr.io/sidecar-memory-limit: "256Mi"
    spec:
      containers:
        - name: order-service
          image: my-registry/order-service-rust:v1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 3
```

## Health Check in Actix-web

```rust
use actix_web::{web, App, HttpServer, HttpResponse};

async fn health() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({"status": "ok"}))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/healthz", web::get().to(health))
            .route("/readyz", web::get().to(health))
            // ... other routes
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## Deploying to Kubernetes

```bash
# Install Dapr on the cluster
dapr init --kubernetes --wait

# Apply components and deployment
kubectl apply -f k8s/components/
kubectl apply -f k8s/deployment.yaml

# Verify pod is running with Dapr sidecar
kubectl get pods -l app=order-service-rust
kubectl describe pod <pod-name>
kubectl logs <pod-name> -c daprd
```

## Horizontal Pod Autoscaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service-rust
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Summary

Rust's small binary size and `scratch`-based images result in extremely lightweight container images. On Kubernetes, Dapr sidecar injection is triggered by the `dapr.io/enabled: "true"` annotation, and resource limits keep sidecar overhead minimal. The combination of Rust's low memory footprint and Dapr's infrastructure abstractions makes for highly efficient Kubernetes deployments.
