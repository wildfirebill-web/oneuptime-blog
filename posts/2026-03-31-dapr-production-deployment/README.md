# How to Prepare a Dapr Deployment for Production

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Production, Kubernetes, Deployment, Security

Description: Learn how to harden a Dapr deployment for production on Kubernetes with mTLS, secrets management, resiliency policies, observability, and resource limits.

---

## Production Dapr Is Different from Development

`dapr init` gives you a local environment with Redis and Zipkin. Production requires hardening: mutual TLS between sidecars, proper secrets management (not env vars), resource limits on sidecars, observability integration, and network policies. This post walks through each concern.

## Installing Dapr on Kubernetes

Use Helm for production installs - it gives you configuration control:

```bash
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update

helm upgrade --install dapr dapr/dapr \
  --namespace dapr-system \
  --create-namespace \
  --set global.mtls.enabled=true \
  --set dapr_operator.replicaCount=2 \
  --set dapr_placement.replicaCount=3 \
  --set dapr_sentry.replicaCount=2 \
  --set dapr_dashboard.enabled=false \
  --wait
```

## Enabling mTLS

mTLS is on by default in Kubernetes. Verify it is active:

```bash
kubectl get configuration/daprsystem -n dapr-system -o yaml | grep mtls
```

To configure certificate rotation interval:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
  namespace: production
spec:
  mtls:
    enabled: true
    workloadCertTTL: "24h"
    allowedClockSkew: "15m"
```

## Sidecar Resource Limits

Always set resource limits on Dapr sidecars to prevent noisy-neighbor issues:

```yaml
# deployment.yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "order-service"
  dapr.io/app-port: "5000"
  dapr.io/sidecar-cpu-request: "100m"
  dapr.io/sidecar-cpu-limit: "300m"
  dapr.io/sidecar-memory-request: "64Mi"
  dapr.io/sidecar-memory-limit: "256Mi"
  dapr.io/log-level: "warn"
```

## Production State Store with Redis Sentinel

```yaml
# components/production/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: production
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-sentinel:26379"
  - name: sentinelMasterName
    value: "mymaster"
  - name: redisPassword
    secretKeyRef:
      name: redis-password
      key: password
  - name: enableTLS
    value: "true"
  - name: maxRetries
    value: "3"
auth:
  secretStore: kubernetes
```

## Production Resiliency Policies

```yaml
# components/production/resiliency.yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: production-resiliency
  namespace: production
spec:
  policies:
    retries:
      DefaultRetry:
        policy: exponential
        maxRetries: 5
        maxInterval: 30s
        maxElapsedTime: 2m
    timeouts:
      DefaultTimeout: 10s
    circuitBreakers:
      DefaultCircuitBreaker:
        maxRequests: 5
        interval: 30s
        timeout: 60s
        trip: consecutiveFailures >= 5

  targets:
    components:
      statestore:
        retry: DefaultRetry
        timeout: DefaultTimeout
        circuitBreaker: DefaultCircuitBreaker
```

## Observability: OpenTelemetry Export

```yaml
# components/production/tracing.yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
  namespace: production
spec:
  tracing:
    samplingRate: "0.1"   # 10% sampling in production
    otel:
      endpointAddress: "http://otel-collector:4318"
      isSecure: false
      protocol: http
  metric:
    enabled: true
```

## Network Policies

Restrict sidecar communication to only allow Dapr control-plane traffic:

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dapr-sidecar-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      dapr.io/enabled: "true"
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: dapr-system
    ports:
    - protocol: TCP
      port: 3500
    - protocol: TCP
      port: 50001
```

## Summary

A production Dapr deployment on Kubernetes requires Helm-managed installation with multiple replicas for control-plane components, explicit sidecar resource limits, mTLS enabled, secrets stored in Kubernetes Secrets (not env vars), resiliency policies protecting all external component calls, and OpenTelemetry export for traces and metrics. These steps together produce a deployment that is secure, observable, and resilient.
