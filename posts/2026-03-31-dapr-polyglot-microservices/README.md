# How to Use Dapr in a Polyglot Microservices Environment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Microservice, Polyglot, Kubernetes, Service Invocation

Description: Learn how to use Dapr to connect microservices written in different languages, enabling seamless communication without language-specific SDKs.

---

Polyglot architectures let teams choose the best language for each service - Python for ML, Go for high-throughput APIs, Node.js for real-time features. Dapr provides a unified sidecar model that lets every service speak the same API regardless of implementation language.

## How Dapr Enables Polyglot Communication

Dapr injects a sidecar proxy alongside each service. Every service calls its local sidecar via HTTP or gRPC, and the sidecar handles discovery, retries, and observability. This means a Python service can invoke a Go service using the exact same API pattern.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
    spec:
      containers:
      - name: order-service
        image: myrepo/order-service:latest
```

## Python Service Calling a Go Service

```python
import requests

# Python service calling Go service via Dapr sidecar
def get_inventory(product_id: str):
    url = f"http://localhost:3500/v1.0/invoke/inventory-service/method/products/{product_id}"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()
```

The Go inventory service handles the request normally:

```go
package main

import (
    "encoding/json"
    "net/http"
    "github.com/gorilla/mux"
)

func getProduct(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    productID := vars["id"]
    product := fetchProduct(productID)
    json.NewEncoder(w).Encode(product)
}

func main() {
    r := mux.NewRouter()
    r.HandleFunc("/products/{id}", getProduct).Methods("GET")
    http.ListenAndServe(":8080", r)
}
```

## Using Pub/Sub Across Languages

Dapr pub/sub abstracts the message broker. A Node.js service can publish and a Java service can subscribe.

```javascript
// Node.js publisher
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();

async function publishOrder(order) {
    await client.pubsub.publish('pubsub', 'orders', order);
    console.log('Order published:', order.id);
}
```

```java
// Java subscriber (Spring Boot)
@PostMapping("/orders")
@Topic(name = "orders", pubsubName = "pubsub")
public ResponseEntity<Void> handleOrder(@RequestBody Order order) {
    orderService.process(order);
    return ResponseEntity.ok().build();
}
```

## Shared State Across Language Boundaries

Any service can read or write shared state regardless of language:

```bash
# From any service via HTTP
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "session-123", "value": {"userId": "abc", "cart": []}}]'

# Read from another language
curl http://localhost:3500/v1.0/state/statestore/session-123
```

## Configuring Dapr Components Once for All Services

You define components once in YAML and every service gets access:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis-master:6379
```

## Summary

Dapr's sidecar model eliminates language-specific integration complexity, letting polyglot teams share state, messaging, and service discovery through a uniform HTTP/gRPC API. Define components once, and every service - regardless of language - gains access to the same infrastructure building blocks. This approach reduces boilerplate across services and makes adding new languages to your stack nearly effortless.
