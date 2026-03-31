# How to Use Dapr Dubbo Output Binding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, Dubbo, RPC, Microservice

Description: Learn how to configure the Dapr Dubbo output binding to invoke Apache Dubbo RPC services from Dapr-enabled microservices in a polyglot environment.

---

## Overview of the Dapr Dubbo Binding

Apache Dubbo is a high-performance Java RPC framework widely used in enterprise Java microservices. The Dapr Dubbo output binding enables non-Java services to call Dubbo RPC endpoints through Dapr's uniform binding API, without writing Dubbo client code.

## Prerequisites

You need a running Dubbo service registered with a ZooKeeper or Nacos registry. This example uses ZooKeeper.

## Start ZooKeeper and a Sample Dubbo Service

```bash
# Start ZooKeeper
docker run -d \
  --name zookeeper \
  -p 2181:2181 \
  zookeeper:3.8

# Deploy your Dubbo service (example Spring Boot app)
# The service should register itself with ZooKeeper on startup
```

## Configure the Dubbo Output Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: dubbo-service
spec:
  type: bindings.dubbo
  version: v1
  metadata:
  - name: registryAddress
    value: zookeeper://localhost:2181
  - name: registryMaxRetries
    value: "3"
  - name: timeout
    value: "5000"
  - name: version
    value: "1.0.0"
  - name: group
    value: mygroup
```

## Invoke a Dubbo Service Method

```bash
curl -X POST http://localhost:3500/v1.0/bindings/dubbo-service \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "create",
    "data": {
      "methodName": "getUserById",
      "args": ["user-123"],
      "argTypes": ["java.lang.String"]
    },
    "metadata": {
      "interfaceName": "com.example.UserService",
      "version": "1.0.0"
    }
  }'
```

## Invoke with Complex Parameters

```bash
curl -X POST http://localhost:3500/v1.0/bindings/dubbo-service \
  -H "Content-Type: application/json" \
  -d '{
    "operation": "create",
    "data": {
      "methodName": "createOrder",
      "args": [
        {
          "customerId": "cust-456",
          "productId": "prod-789",
          "quantity": 3
        }
      ],
      "argTypes": ["com.example.dto.OrderRequest"]
    },
    "metadata": {
      "interfaceName": "com.example.OrderService"
    }
  }'
```

## Application Code Integration

```python
import requests

def call_dubbo_service(interface: str, method: str, args: list, arg_types: list):
    response = requests.post(
        "http://localhost:3500/v1.0/bindings/dubbo-service",
        json={
            "operation": "create",
            "data": {
                "methodName": method,
                "args": args,
                "argTypes": arg_types,
            },
            "metadata": {
                "interfaceName": interface,
            },
        },
    )
    response.raise_for_status()
    return response.json()

# Fetch user from Java Dubbo service
user = call_dubbo_service(
    interface="com.example.UserService",
    method="getUserById",
    args=["user-123"],
    arg_types=["java.lang.String"],
)
print("User:", user)
```

## Multiple Dubbo Services

Define separate binding components for each Dubbo interface:

```yaml
# user-service-binding.yaml
metadata:
  name: user-dubbo-service
spec:
  type: bindings.dubbo
  metadata:
  - name: registryAddress
    value: zookeeper://localhost:2181
```

```yaml
# order-service-binding.yaml
metadata:
  name: order-dubbo-service
spec:
  type: bindings.dubbo
  metadata:
  - name: registryAddress
    value: zookeeper://localhost:2181
```

## Summary

The Dapr Dubbo output binding enables polyglot services to call Apache Dubbo RPC endpoints without Dubbo client libraries. Configure the ZooKeeper or Nacos registry address in the component, then invoke Dubbo service methods by specifying the interface name, method name, arguments, and Java type signatures in the binding payload.
