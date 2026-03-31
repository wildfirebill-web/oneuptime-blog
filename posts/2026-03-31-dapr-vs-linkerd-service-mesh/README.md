# Dapr vs Linkerd: Application Runtime vs Service Mesh

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Linkerd, Service Mesh, Microservice, Comparison

Description: Compare Dapr and Linkerd to understand their complementary roles - Dapr as an application runtime and Linkerd as a lightweight service mesh.

---

Dapr and Linkerd are both sidecar-based tools for Kubernetes microservices, but their sidecar containers serve entirely different purposes. Understanding their distinct roles helps you decide which to adopt and whether to use both.

## What Linkerd Does

Linkerd is a lightweight, security-focused service mesh. Its ultra-small Rust-based proxy (linkerd2-proxy) intercepts all TCP traffic between services, providing:

- Automatic mutual TLS between all meshed services
- Latency-aware load balancing (EWMA algorithm)
- Real-time traffic metrics (success rate, latency, throughput)
- Retry and timeout policies
- Traffic splitting for canary deployments

Linkerd requires zero application changes - it works purely at the network level.

## What Dapr Does

Dapr is a distributed application runtime. Applications call Dapr's APIs explicitly to use building blocks:

```python
# Application explicitly calls Dapr's state API
with DaprClient() as d:
    d.save_state(store_name='statestore', key='order-123', value=order)
    result = d.get_state(store_name='statestore', key='order-123')
```

Your application is aware of and depends on Dapr's presence.

## Key Architectural Difference

| Aspect | Linkerd | Dapr |
|--------|---------|------|
| App awareness | Transparent | Explicit API calls |
| Primary concern | Network reliability | Application building blocks |
| Proxy language | Rust (tiny footprint) | Go |
| mTLS | Automatic for all traffic | Between Dapr sidecars |
| State management | No | Yes |
| Messaging | No | Yes (pub/sub) |
| Actors | No | Yes |

## When Linkerd Wins

Choose Linkerd when you need reliability and security improvements without changing application code. Linkerd's zero-config mTLS and latency-aware load balancing improve any service mesh, regardless of language or framework.

```bash
# Inject Linkerd into a namespace - zero app changes needed
kubectl annotate namespace production \
  linkerd.io/inject=enabled
```

## When Dapr Wins

Choose Dapr when your application needs portable access to infrastructure. Dapr's building blocks let you swap Redis for DynamoDB for Cosmos DB by changing a YAML file - your application code stays the same.

## Using Both Together

Both tools can run simultaneously. Use Linkerd for transparent mTLS and observability across all services, and Dapr for application-level building blocks. Disable Dapr's built-in mTLS to avoid redundancy:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  mtls:
    enabled: false
```

Linkerd then secures the Dapr sidecar-to-sidecar traffic along with all other traffic in the mesh.

## Summary

Linkerd and Dapr serve different purposes. Linkerd provides transparent, lightweight service mesh capabilities without application changes. Dapr provides explicit building block APIs for state, messaging, and workflows. They are complementary - use Linkerd for network-level reliability and Dapr for application-level distributed systems patterns, with Dapr's mTLS disabled when both are present.
