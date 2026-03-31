# How to Configure NameFormat Name Resolution in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Name Resolution, NameFormat, Service Discovery, Configuration

Description: Learn how to configure Dapr's NameFormat name resolution component to map app IDs to custom hostname patterns using template-based formatting.

---

## What Is NameFormat Name Resolution?

The `nameresolution.nameformat` component in Dapr allows you to define a custom hostname pattern that maps Dapr app IDs to network addresses. Instead of relying on dynamic discovery (mDNS, Consul, etc.), NameFormat uses a Go template to construct the target hostname from the app ID.

This is useful in environments where service hostnames follow predictable patterns, such as `{appid}.service.internal` or `{appid}.namespace.svc.cluster.local`.

## Basic Configuration

Create a NameFormat name resolution component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nameresolution
spec:
  type: nameresolution.nameformat
  version: v1
  metadata:
    - name: nameFormat
      value: "{{"{{"}} .ID {{"}}"}}.services.internal"
```

With this configuration, an app ID of `order-service` resolves to `order-service.services.internal`.

## Using Template Variables

The template supports the following variables:

- `.ID` - the Dapr app ID
- `.Namespace` - the app's namespace
- `.Port` - the Dapr sidecar port

Example with namespace:

```yaml
metadata:
  - name: nameFormat
    value: "{{"{{"}} .ID {{"}}"}}.{{"{{"}} .Namespace {{"}}"}}.svc.cluster.local"
```

An invocation of `order-service` in namespace `production` resolves to:
`order-service.production.svc.cluster.local`

## Custom DNS Subdomain Example

For an environment where each service has a DNS entry in a custom subdomain:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nameresolution
spec:
  type: nameresolution.nameformat
  version: v1
  metadata:
    - name: nameFormat
      value: "dapr-{{"{{"}} .ID {{"}}"}}.internal.example.com"
```

With this pattern:
- `order-service` resolves to `dapr-order-service.internal.example.com`
- `payment-service` resolves to `dapr-payment-service.internal.example.com`

Ensure your DNS server has entries for each service matching this pattern.

## Applying the Component

Place the component in your Dapr components directory:

```bash
cp nameresolution-nameformat.yaml ~/.dapr/components/
```

Or reference it via `--components-path`:

```bash
dapr run --app-id order-service \
  --app-port 8080 \
  --components-path ./components \
  -- ./order-service
```

For Kubernetes:

```bash
kubectl apply -f nameresolution-nameformat.yaml
```

## Testing NameFormat Resolution

Start a service and verify it can resolve others:

```bash
dapr invoke --app-id payment-service \
  --method health \
  --verb GET
```

If resolution fails, check the resolved hostname manually:

```bash
nslookup dapr-payment-service.internal.example.com
```

Enable debug logging for more detail:

```bash
dapr run --app-id myapp --log-level debug \
  --components-path ./components -- ./myapp 2>&1 | grep -i resolve
```

## Comparison with Other Name Resolution Components

| Component | Best For |
|-----------|----------|
| `mdns` | Local development, same host |
| `kubernetes` | Kubernetes clusters |
| `consul` | Multi-cloud, bare metal |
| `sqlite` | Docker Compose, single host |
| `nameformat` | Predictable DNS patterns |

## Summary

The NameFormat name resolution component maps Dapr app IDs to hostnames using Go templates. It is ideal for environments with predictable service hostname patterns. Configure the `nameFormat` template with app ID, namespace, and port variables to match your infrastructure's DNS naming convention.
