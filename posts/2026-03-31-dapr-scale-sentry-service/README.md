# How to Scale Dapr Sentry Service

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sentry, Scaling, Certificate, High Availability

Description: Learn how to scale the Dapr Sentry service for high availability and increased certificate issuance throughput in production Kubernetes clusters.

---

The Dapr Sentry service is the certificate authority for Dapr's mTLS communication. Every sidecar in your cluster obtains workload certificates from Sentry. As your cluster grows, Sentry may become a bottleneck during mass restarts or deployments. Scaling Sentry horizontally increases certificate issuance capacity and removes it as a single point of failure.

## Sentry Scaling Model

Sentry is stateless with respect to certificate issuance. It reads the CA from the `dapr-trust-bundle` secret and signs certificate requests on demand. Multiple replicas can run concurrently - sidecars will connect to any available Sentry replica via the Kubernetes service. There is no leader election or coordination needed between replicas.

## Scaling with Helm

Increase the Sentry replica count using Helm:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_sentry.replicaCount=3 \
  --reuse-values
```

Verify all replicas come up:

```bash
kubectl rollout status deployment/dapr-sentry -n dapr-system
kubectl get pods -n dapr-system -l app=dapr-sentry
```

## Setting Resource Requests and Limits

Certificate signing is CPU-intensive. Set appropriate resources for Sentry under load:

```yaml
dapr_sentry:
  replicaCount: 3
  resources:
    requests:
      cpu: 100m
      memory: 64Mi
    limits:
      cpu: 500m
      memory: 256Mi
```

Apply with Helm:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  --set dapr_sentry.replicaCount=3 \
  --set "dapr_sentry.resources.requests.cpu=100m" \
  --set "dapr_sentry.resources.requests.memory=64Mi" \
  --set "dapr_sentry.resources.limits.cpu=500m" \
  --set "dapr_sentry.resources.limits.memory=256Mi" \
  --reuse-values
```

## Enabling Pod Anti-Affinity

Spread Sentry replicas across nodes to survive node failures:

```yaml
dapr_sentry:
  replicaCount: 3
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: dapr-sentry
            topologyKey: kubernetes.io/hostname
```

Save this as `sentry-values.yaml` and apply:

```bash
helm upgrade dapr dapr/dapr \
  --namespace dapr-system \
  -f sentry-values.yaml \
  --reuse-values
```

## Configuring Pod Disruption Budget

Prevent all Sentry replicas from being evicted simultaneously during node maintenance:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: dapr-sentry-pdb
  namespace: dapr-system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: dapr-sentry
```

Apply this manifest:

```bash
kubectl apply -f sentry-pdb.yaml
```

## Monitoring Sentry Certificate Issuance Rate

After scaling, verify Sentry is handling certificate requests across all replicas:

```bash
kubectl port-forward svc/dapr-sentry -n dapr-system 9090:9090
curl -s http://localhost:9090/metrics | grep dapr_sentry_cert_sign
```

Key metrics include `dapr_sentry_cert_sign_request_received_total` and `dapr_sentry_cert_sign_success_total`. Confirm all replicas show positive request counts.

## Summary

Scaling Dapr Sentry is straightforward because the service is stateless - multiple replicas can handle certificate signing requests without coordination. Increase replicas with Helm, set CPU-appropriate resource limits, apply pod anti-affinity rules to spread across nodes, and add a PodDisruptionBudget to protect availability during maintenance. Monitor the certificate sign metrics to confirm load is distributed across all replicas.
