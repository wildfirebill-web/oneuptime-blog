# How to Configure JSON Serialization in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, JSON, Serialization, State Management, Pub/Sub

Description: Configure JSON serialization behavior in Dapr state management, service invocation, and pub/sub to control data encoding, null handling, and field naming conventions.

---

## JSON Serialization in Dapr

Dapr uses JSON as the default serialization format for state management and pub/sub payloads. Understanding how Dapr SDKs serialize and deserialize data helps you avoid common issues like null fields, date formatting inconsistencies, and naming convention mismatches across polyglot services.

## JSON in Dapr State Management (.NET)

```csharp
// Configure System.Text.Json options for Dapr
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDaprClient(client =>
{
    client.UseJsonSerializationOptions(new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull,
        Converters =
        {
            new JsonStringEnumConverter(JsonNamingPolicy.CamelCase)
        },
        WriteIndented = false
    });
});
```

## Saving and Loading Typed State

```csharp
public class OrderService
{
    private readonly DaprClient _dapr;
    private const string StateStore = "statestore";

    public async Task SaveOrderAsync(Order order)
    {
        // Dapr serializes Order to JSON automatically
        await _dapr.SaveStateAsync(StateStore, $"order-{order.Id}", order);
    }

    public async Task<Order?> GetOrderAsync(string orderId)
    {
        // Dapr deserializes JSON back to Order
        return await _dapr.GetStateAsync<Order>(StateStore, $"order-{orderId}");
    }
}

// Model with JSON attributes
public record Order(
    [property: JsonPropertyName("id")] string Id,
    [property: JsonPropertyName("customer_id")] string CustomerId,
    [property: JsonPropertyName("status")] OrderStatus Status,
    [property: JsonPropertyName("created_at")] DateTime CreatedAt,
    [property: JsonPropertyName("items")] List<OrderItem> Items
);

public enum OrderStatus { Pending, Processing, Fulfilled, Cancelled }
```

## JSON Serialization in Python SDK

```python
# Python Dapr SDK uses json module by default
import json
from dataclasses import dataclass, asdict
from datetime import datetime
from dapr.clients import DaprClient

@dataclass
class Order:
    order_id: str
    customer_id: str
    amount: float
    created_at: str
    status: str = "pending"

def serialize_order(order: Order) -> bytes:
    """Custom JSON serializer for Dapr state."""
    data = asdict(order)
    return json.dumps(data, ensure_ascii=False).encode('utf-8')

def save_order(order: Order):
    with DaprClient() as client:
        # Save with explicit JSON serialization
        client.save_state(
            store_name="statestore",
            key=f"order-{order.order_id}",
            value=json.dumps(asdict(order)),
            state_metadata={"contentType": "application/json"}
        )

def get_order(order_id: str) -> Order:
    with DaprClient() as client:
        result = client.get_state(
            store_name="statestore",
            key=f"order-{order_id}"
        )
        data = json.loads(result.data)
        return Order(**data)
```

## JSON in Pub/Sub with CloudEvents

```javascript
// Node.js - Custom JSON handling in pub/sub
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();

// Publishing with custom JSON payload
async function publishOrder(order) {
  // Dapr wraps this in a CloudEvent envelope
  await client.pubsub.publish('pubsub', 'order-created', {
    orderId: order.id,
    customerId: order.customerId,
    amount: order.amount,
    // ISO 8601 for cross-service date compatibility
    createdAt: new Date().toISOString(),
    items: order.items.map(item => ({
      productId: item.productId,
      quantity: item.quantity,
      unitPrice: item.unitPrice
    }))
  });
}

// Subscribing and parsing
app.post('/events/order-created', (req, res) => {
  const event = req.body; // CloudEvent
  const order = event.data; // Your JSON payload

  console.log('Order ID:', order.orderId);
  console.log('Created At:', new Date(order.createdAt));

  res.sendStatus(200);
});
```

## Handling JSON Schema Mismatches

```yaml
# Use Dapr middleware for JSON transformation
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: transform-middleware
spec:
  type: middleware.http.transformer
  version: v1
  metadata:
  - name: rule
    value: |
      .data | {
        order_id: .orderId,
        customer_id: .customerId
      }
```

## Summary

Dapr uses JSON as its default serialization format, and each SDK exposes configuration hooks to customize field naming, null handling, and enum encoding. Aligning JSON serialization settings across polyglot services prevents data loss and field mismatches. For pub/sub, ensure all services agree on date formats (ISO 8601) and field naming conventions (camelCase or snake_case) by establishing shared schema contracts.
