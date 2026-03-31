# How to Use Dapr Service Invocation Between Java and Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Java, Node.js, Service Invocation, Microservice

Description: Invoke a Node.js Express service from a Java Spring Boot application using Dapr service invocation, with examples for both HTTP and the Java Dapr SDK.

---

Dapr's service invocation API makes calling a Node.js service from Java as simple as an HTTP call to localhost. The Dapr sidecar handles service discovery, mTLS, and retries automatically.

## Node.js Service (Target)

Create a Node.js Express service that processes product lookups:

```javascript
// server.js
const express = require('express');
const app = express();
app.use(express.json());

const products = {
  'prod-001': { name: 'Widget', price: 9.99, inStock: true },
  'prod-002': { name: 'Gadget', price: 24.99, inStock: false }
};

app.get('/products/:id', (req, res) => {
  const product = products[req.params.id];
  if (!product) {
    return res.status(404).json({ error: 'Product not found' });
  }
  res.json(product);
});

app.listen(3000, () => console.log('Product service on port 3000'));
```

Kubernetes annotations:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "product-service-node"
  dapr.io/app-port: "3000"
```

## Java Spring Boot Service (Caller)

Add the Dapr Spring Boot SDK dependency:

```xml
<dependency>
    <groupId>io.dapr</groupId>
    <artifactId>dapr-sdk-springboot</artifactId>
    <version>1.12.0</version>
</dependency>
```

Create a service class using the Dapr client:

```java
import io.dapr.client.DaprClient;
import io.dapr.client.DaprClientBuilder;
import io.dapr.client.domain.HttpExtension;
import org.springframework.stereotype.Service;

@Service
public class ProductServiceClient {

    private final DaprClient daprClient;

    public ProductServiceClient() {
        this.daprClient = new DaprClientBuilder().build();
    }

    public Product getProduct(String productId) {
        return daprClient.invokeMethod(
            "product-service-node",     // Target app-id
            "products/" + productId,    // Method path
            null,                       // Request body (GET has none)
            HttpExtension.GET,
            Product.class
        ).block();
    }
}
```

## REST Controller Using the Client

```java
@RestController
@RequestMapping("/api")
public class OrderController {

    @Autowired
    private ProductServiceClient productClient;

    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        Product product = productClient.getProduct(request.getProductId());

        if (!product.isInStock()) {
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(null);
        }

        Order order = new Order(request.getProductId(), product.getPrice());
        return ResponseEntity.ok(order);
    }
}
```

## Using Raw HTTP Instead of SDK

For simpler cases, call the Dapr sidecar HTTP API directly from Java:

```java
import org.springframework.web.client.RestTemplate;

String daprPort = System.getenv().getOrDefault("DAPR_HTTP_PORT", "3500");
String url = "http://localhost:" + daprPort
    + "/v1.0/invoke/product-service-node/method/products/" + productId;

RestTemplate restTemplate = new RestTemplate();
Product product = restTemplate.getForObject(url, Product.class);
```

## Testing Cross-Language Invocation

```bash
# Port-forward the Java service
kubectl port-forward deployment/order-service-java 8080:8080

# Create an order (Java -> Dapr -> Node.js)
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"productId": "prod-001", "quantity": 2}'
```

## Summary

Dapr service invocation between Java and Node.js uses the app ID as the service locator, eliminating hardcoded URLs and service discovery logic. The Java Spring Boot application uses the Dapr Java SDK for typed calls, while the Node.js service exposes standard HTTP endpoints. Both services remain unaware of each other's location or infrastructure.
