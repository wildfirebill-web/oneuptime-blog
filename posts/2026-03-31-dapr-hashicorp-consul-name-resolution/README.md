# How to Configure HashiCorp Consul Name Resolution in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Consul, Name Resolution, Service Discovery, HashiCorp

Description: Learn how to configure Dapr to use HashiCorp Consul for service discovery and name resolution in self-hosted and Kubernetes environments.

---

## Why Use Consul with Dapr?

HashiCorp Consul provides robust service discovery, health checking, and key-value storage. When running Dapr in environments where Kubernetes DNS is unavailable (multi-cloud, bare metal, or hybrid deployments), Consul provides a reliable alternative for name resolution. Dapr's `nameresolution.consul` component registers services with Consul and resolves app IDs through Consul's service catalog.

## Setting Up Consul

Start a local Consul agent for development:

```bash
consul agent -dev -bind=127.0.0.1
```

For production, deploy Consul as a cluster. Using Docker Compose:

```yaml
version: "3.8"
services:
  consul:
    image: hashicorp/consul:1.17
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    command: "agent -server -bootstrap-expect=1 -ui -client=0.0.0.0 -bind=0.0.0.0"
```

Start it:

```bash
docker compose up -d consul
```

## Configuring the Dapr Consul Component

Create a Dapr component for Consul name resolution:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nameresolution
spec:
  type: nameresolution.consul
  version: v1
  metadata:
    - name: client
      value: |
        {
          "address": "127.0.0.1:8500",
          "scheme": "http",
          "datacenter": "dc1"
        }
    - name: checks
      value: |
        [
          {
            "name": "Dapr Health Status",
            "checkID": "daprHealth",
            "interval": "15s",
            "http": "http://localhost:3500/v1.0/healthz"
          }
        ]
    - name: tags
      value: |
        ["dapr"]
    - name: queryOptions
      value: |
        {
          "useCache": true
        }
```

## Starting Dapr with Consul

Place the component file in `~/.dapr/components/` or point to the directory:

```bash
dapr run --app-id order-service \
  --app-port 8080 \
  --components-path ./components \
  -- ./order-service
```

Dapr will register the service in Consul on startup. Verify registration:

```bash
curl http://localhost:8500/v1/catalog/services | jq .
```

You should see `order-service` in the service list.

## Service Invocation Through Consul

Once registered, invoke services by app ID as usual:

```bash
curl http://localhost:3500/v1.0/invoke/payment-service/method/pay
```

Dapr resolves `payment-service` by querying Consul's service catalog.

## ACL Token Authentication

For secure Consul clusters with ACLs enabled:

```yaml
metadata:
  - name: client
    value: |
      {
        "address": "consul.example.com:8500",
        "scheme": "https",
        "token": "my-consul-acl-token"
      }
```

Store the token as a Kubernetes secret and reference it:

```yaml
  - name: client
    secretKeyRef:
      name: consul-secret
      key: acl-token
```

## Health Checks and Deregistration

Consul removes unhealthy services automatically. The health check configuration in the component ensures Dapr reports its health status. If the app restarts, Dapr re-registers with Consul on startup.

To manually deregister a service:

```bash
curl -X PUT http://localhost:8500/v1/agent/service/deregister/order-service
```

## Summary

Dapr's Consul name resolution component registers services with HashiCorp Consul and resolves app IDs through the service catalog. It is ideal for non-Kubernetes environments and hybrid deployments. Configure health checks to enable automatic deregistration of failed services, and use ACL tokens for secure Consul clusters.
