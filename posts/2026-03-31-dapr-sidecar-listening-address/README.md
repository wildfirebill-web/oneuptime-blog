# How to Configure Dapr Sidecar Listening Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Sidecar, Networking, Kubernetes, Configuration

Description: Configure the network interface and ports the Dapr sidecar listens on to control API accessibility, enable dual-stack networking, or restrict access to localhost only.

---

By default, the Dapr sidecar listens on all network interfaces (`0.0.0.0`) on its HTTP (3500) and gRPC (50001) ports. In some security-sensitive environments, you may want to restrict the sidecar to a specific interface, change the default ports, or bind only to localhost.

## Default Listening Behavior

When injected into a pod, daprd binds to:
- HTTP API: `0.0.0.0:3500`
- gRPC API: `0.0.0.0:50001`
- Internal gRPC: `0.0.0.0:50002`
- Public gRPC (for service invocation): `0.0.0.0:3501`

This means the sidecar is accessible from other processes in the pod on any interface.

## Changing the HTTP and gRPC Ports

If your application's ports conflict with Dapr's defaults, change them via annotations:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "api-service"
  dapr.io/http-port: "3600"
  dapr.io/grpc-port: "50101"
```

Ensure your application code uses the updated port when calling Dapr APIs:

```javascript
const DAPR_PORT = process.env.DAPR_HTTP_PORT || 3600;
const response = await fetch(`http://localhost:${DAPR_PORT}/v1.0/state/statestore/my-key`);
```

## Restricting to Localhost

To prevent other pods from directly calling the sidecar API (relying on mTLS service invocation instead), bind only to localhost:

```yaml
annotations:
  dapr.io/sidecar-listen-addresses: "127.0.0.1"
```

With this setting, only processes within the same container can reach the Dapr HTTP and gRPC APIs directly.

## Enabling IPv6 or Dual-Stack

For clusters with IPv6 or dual-stack networking:

```yaml
annotations:
  dapr.io/sidecar-listen-addresses: "0.0.0.0,[::]"
```

This binds to both IPv4 and IPv6 interfaces.

## Internal vs. Public Ports

Dapr uses different ports for internal and external communication:

```bash
# Check which ports daprd is listening on
kubectl exec my-pod -c daprd -- ss -tlnp
```

Expected output:

```text
State  Recv-Q  Send-Q  Local Address:Port
LISTEN 0       128     0.0.0.0:3500
LISTEN 0       128     0.0.0.0:3501
LISTEN 0       128     0.0.0.0:50001
LISTEN 0       128     0.0.0.0:50002
```

## App-to-Sidecar Port

Your application calls the sidecar on port 3500 (HTTP) or 50001 (gRPC). These are always on localhost from the application's perspective since they share a pod network namespace.

```python
import requests
resp = requests.get("http://localhost:3500/v1.0/state/statestore/order-123")
```

## Summary

Configuring the Dapr sidecar listening address lets you harden security by restricting API access to localhost, avoid port conflicts with your application, and support dual-stack IPv6 networking. Most production deployments use the defaults, but localhost-only binding combined with mTLS provides an additional security boundary.
