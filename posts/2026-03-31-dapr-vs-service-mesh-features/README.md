# How to Choose Between Dapr and Service Mesh Features

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Service Mesh, Architecture, Microservice, Kubernetes

Description: Understand the feature overlap between Dapr and service meshes like Istio to make informed decisions about which tool to use for each concern.

---

Dapr and service meshes like Istio solve overlapping problems. Both provide service discovery, mTLS, observability, and resiliency. However, they operate at different layers and have distinct strengths. Choosing the right tool for each feature prevents unnecessary complexity and double overhead.

## Layer of Operation

**Dapr** operates at the application layer (L7). It provides APIs your application code calls directly - state management, pub/sub, service invocation, secrets, and actors. It understands your business logic context.

**Service meshes** operate at the network layer (L4/L7). They intercept TCP and HTTP traffic transparently without application code changes. They do not know about your business entities.

## Feature Comparison

| Feature | Dapr | Service Mesh |
|---------|------|--------------|
| Service invocation | Yes - app-id based | Yes - DNS/load balancing |
| mTLS | Yes - between sidecars | Yes - between all pods |
| Retries | Yes - resiliency policies | Yes - VirtualService |
| Circuit breaker | Yes - resiliency policies | Yes - DestinationRule |
| Observability | Yes - metrics, traces, logs | Yes - metrics, traces |
| Traffic splitting | No | Yes - weighted routing |
| Header-based routing | No | Yes - match rules |
| Rate limiting | Limited | Yes - EnvoyFilter |
| State management | Yes - unique to Dapr | No |
| Pub/Sub | Yes - unique to Dapr | No |
| Actor model | Yes - unique to Dapr | No |
| Secrets API | Yes - unique to Dapr | No |

## Use Dapr For

- **State management**: Storing and retrieving application state with consistency guarantees.
- **Pub/Sub messaging**: Decoupled event-driven communication between services.
- **Service invocation with resiliency**: Calling services with built-in retry and circuit breaker policies.
- **Actor model**: Managing stateful per-entity logic with concurrency safety.
- **Secrets**: Abstracting secret access from your code.

```yaml
# Dapr resiliency policy - use this for application-level retry logic
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: payment-resiliency
spec:
  policies:
    retries:
      retryForever:
        policy: exponential
        maxRetries: 5
```

## Use Service Mesh For

- **Traffic splitting and canary releases**: Percentage-based routing between versions.
- **mTLS enforcement cluster-wide**: Encrypting all pod-to-pod traffic automatically.
- **Ingress/egress control**: Controlling traffic entering and leaving the mesh.
- **Rate limiting**: Enforcing per-service request quotas.

```yaml
# Istio - use this for network-level traffic control
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: checkout
spec:
  hosts:
  - checkout
  http:
  - route:
    - destination:
        host: checkout
        subset: v1
      weight: 95
    - destination:
        host: checkout
        subset: v2
      weight: 5
```

## Avoid Using Both For the Same Thing

When you have both Dapr and a service mesh:

- **Use only one for mTLS** - prefer the service mesh for cluster-wide mTLS, disable Dapr mTLS.
- **Use only one for retries** - pick Dapr resiliency or mesh retry policies, not both (double retries amplify load).
- **Use only one for circuit breaking** - Dapr resiliency or DestinationRule outlier detection.

## Summary

Dapr excels at application-level abstractions - state, pub/sub, actors, and secrets - that service meshes do not provide. Service meshes excel at network-level controls - traffic splitting, cluster-wide mTLS, and rate limiting - that Dapr does not support. When both are deployed, delineate responsibilities clearly: use Dapr for application APIs and the mesh for network policy, avoiding feature duplication that adds overhead without benefit.
