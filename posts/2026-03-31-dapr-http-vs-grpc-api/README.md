# How to Choose Between Dapr HTTP and gRPC APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, HTTP, gRPC, API, Service Invocation

Description: Choose between Dapr's HTTP and gRPC APIs based on performance needs, language support, streaming requirements, and operational simplicity for your microservices.

---

Dapr exposes all its building blocks through both HTTP and gRPC APIs. Both are fully supported, but each has trade-offs in performance, tooling, and operational complexity.

## When to Use the HTTP API

The HTTP API is the simpler choice for most applications:

```bash
# Invoke a service via HTTP
curl -X POST http://localhost:3500/v1.0/invoke/target-service/method/orders \
  -H "Content-Type: application/json" \
  -d '{"order_id": 42}'

# Save state via HTTP
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key":"user-1","value":{"name":"Alice"}}]'
```

Choose HTTP when:
- Your team is more familiar with REST/HTTP debugging
- You need easy integration with HTTP-native tools (Postman, curl)
- Your application already uses an HTTP client
- You are using a language without a mature Dapr SDK

## When to Use the gRPC API

gRPC offers lower latency and better throughput for high-frequency calls:

```python
# Python gRPC service invocation
from dapr.clients import DaprClient

with DaprClient() as client:
    response = client.invoke_method(
        app_id="target-service",
        method_name="getOrder",
        data=b'{"order_id": 42}',
        content_type="application/json",
        http_verb="POST"
    )
    print(response.data)
```

Choose gRPC when:
- Service-to-service call latency is a bottleneck
- You need streaming (server-side or bidirectional)
- You are using Protobuf schemas for strong typing
- Your application already uses gRPC

## Performance Comparison

A rough benchmark on typical Kubernetes infrastructure:

| Metric | HTTP | gRPC |
|--------|------|------|
| Latency (p50) | ~2-5ms | ~0.5-2ms |
| Throughput | Moderate | High |
| Payload size | Higher (JSON) | Lower (Protobuf) |
| Connection overhead | Per-request | Persistent |

## Using gRPC with Dapr Proxy

Configure your app to receive gRPC calls from Dapr:

```yaml
# In Kubernetes deployment
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "my-service"
  dapr.io/app-port: "50051"
  dapr.io/app-protocol: "grpc"
```

## Mixing HTTP and gRPC

You can use HTTP for some building blocks and gRPC for others. For example, use gRPC for high-throughput service invocation but HTTP for state management operations that happen infrequently:

```javascript
// HTTP for low-frequency state saves
const httpClient = new DaprClient({ daprHost: "127.0.0.1", daprPort: "3500", communicationProtocol: CommunicationProtocolEnum.HTTP });

// gRPC for high-frequency service calls
const grpcClient = new DaprClient({ daprHost: "127.0.0.1", daprPort: "50001", communicationProtocol: CommunicationProtocolEnum.GRPC });
```

## Summary

HTTP is the right default for most Dapr applications due to its simplicity, debugging ease, and broad tooling support. Switch to gRPC when service invocation latency or throughput is a measurable bottleneck, when you need streaming, or when you are already using Protobuf schemas. Both APIs are fully supported and can be mixed within the same application.
