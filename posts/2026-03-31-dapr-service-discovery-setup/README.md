# How to Set Up Service Discovery with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Service Discovery, Kubernetes, DNS, Name Resolution

Description: Learn how Dapr service discovery works, how to configure name resolution components, and how to set up custom discovery for self-hosted and Kubernetes environments.

---

## How Dapr Service Discovery Works

Dapr abstracts service discovery through its name resolution building block. When you invoke `order-service` via Dapr, the sidecar uses a name resolution component to translate the app ID into a network address and port.

Dapr ships with built-in name resolution for:
- Kubernetes (uses kube-dns)
- Self-hosted (uses mDNS for local network)
- HashiCorp Consul

## Default Name Resolution in Kubernetes

No configuration is needed for Kubernetes. Dapr uses the Kubernetes DNS system automatically. When a sidecar calls `order-service`, Dapr resolves it to the Kubernetes service `order-service.namespace.svc.cluster.local`.

Ensure your Dapr-enabled pod has the correct service defined:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 3000
```

## Self-Hosted mDNS Setup

For local development without Kubernetes, Dapr uses mDNS to discover services on the same machine or local network. No additional configuration is needed - just start multiple apps with unique app IDs:

```bash
dapr run --app-id order-service --app-port 3001 -- node order-service.js &
dapr run --app-id inventory-service --app-port 3002 -- node inventory-service.js &
```

Each service will discover the other using mDNS.

## Configuring Consul Name Resolution

For production self-hosted deployments, use Consul for service discovery:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  nameResolution:
    component: consul
    configuration:
      selfRegister: true
      client:
        address: consul-server:8500
      daprPortMetaKey: DAPR_PORT
      queryOptions:
        useCache: true
```

Register services in Consul:

```bash
consul services register -name=order-service -port=3500 -tag=dapr
```

## Testing Service Discovery

```bash
# From any service with a Dapr sidecar, invoke another service by app ID
curl http://localhost:3500/v1.0/invoke/order-service/method/health

# Expected: 200 OK with response from order-service
```

## Troubleshooting Discovery Failures

```bash
# Check registered Dapr apps in self-hosted mode
dapr list

# Check Kubernetes service endpoints
kubectl get endpoints order-service

# Check Dapr sidecar logs for resolution errors
kubectl logs <pod-name> -c daprd | grep "name resolution"
```

## Summary

Dapr service discovery is handled automatically in Kubernetes using kube-dns, and via mDNS for self-hosted local development. For production self-hosted setups, configure the Consul name resolution component. Services are discovered purely by their Dapr app ID without hardcoded hostnames or ports.
