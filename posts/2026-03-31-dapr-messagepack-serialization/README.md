# How to Configure MessagePack Serialization in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, MessagePack, Serialization, Performance, State Management

Description: Use MessagePack binary serialization in Dapr for compact, fast payloads in state management and service invocation, with examples in .NET, Go, and Node.js.

---

## MessagePack vs JSON in Dapr

MessagePack is a binary serialization format that produces payloads 20-50% smaller than equivalent JSON, with faster encode/decode performance. In Dapr, MessagePack is useful for high-frequency state operations, large object graphs in pub/sub, and scenarios where bandwidth or storage costs matter.

```bash
# .NET - Install MessagePack
dotnet add package MessagePack

# Go
go get github.com/vmihailenco/msgpack/v5

# Node.js
npm install @msgpack/msgpack
```

## .NET MessagePack with Dapr State

```csharp
using MessagePack;
using Dapr.Client;

// Define MessagePack-serializable models
[MessagePackObject]
public class Product
{
    [Key(0)]
    public string Id { get; set; } = "";

    [Key(1)]
    public string Name { get; set; } = "";

    [Key(2)]
    public decimal Price { get; set; }

    [Key(3)]
    public int StockCount { get; set; }

    [Key(4)]
    public DateTime UpdatedAt { get; set; }
}

public class ProductStateService
{
    private readonly DaprClient _dapr;
    private const string Store = "statestore";

    public ProductStateService(DaprClient dapr)
    {
        _dapr = dapr;
    }

    public async Task SaveProductAsync(Product product)
    {
        // Serialize to MessagePack binary
        var bytes = MessagePackSerializer.Serialize(product);
        var base64 = Convert.ToBase64String(bytes);

        // Store as base64 string in Dapr state
        await _dapr.SaveStateAsync(Store, $"product-{product.Id}", base64,
            metadata: new Dictionary<string, string>
            {
                ["contentType"] = "application/octet-stream",
                ["encoding"] = "msgpack"
            });
    }

    public async Task<Product?> GetProductAsync(string productId)
    {
        var base64 = await _dapr.GetStateAsync<string>(Store, $"product-{productId}");
        if (base64 == null) return null;

        var bytes = Convert.FromBase64String(base64);
        return MessagePackSerializer.Deserialize<Product>(bytes);
    }
}
```

## Go MessagePack with Dapr

```go
// product_store.go
package main

import (
    "context"
    "fmt"

    dapr "github.com/dapr/go-sdk/client"
    "github.com/vmihailenco/msgpack/v5"
)

type Product struct {
    ID         string  `msgpack:"id"`
    Name       string  `msgpack:"name"`
    Price      float64 `msgpack:"price"`
    StockCount int     `msgpack:"stock_count"`
}

func saveProduct(ctx context.Context, client dapr.Client, p Product) error {
    data, err := msgpack.Marshal(p)
    if err != nil {
        return fmt.Errorf("msgpack marshal failed: %w", err)
    }

    return client.SaveState(ctx, "statestore", "product-"+p.ID, data,
        map[string]string{
            "contentType": "application/octet-stream",
        })
}

func getProduct(ctx context.Context, client dapr.Client, id string) (*Product, error) {
    item, err := client.GetState(ctx, "statestore", "product-"+id, nil)
    if err != nil {
        return nil, err
    }
    if len(item.Value) == 0 {
        return nil, nil
    }

    var p Product
    if err := msgpack.Unmarshal(item.Value, &p); err != nil {
        return nil, fmt.Errorf("msgpack unmarshal failed: %w", err)
    }
    return &p, nil
}
```

## Node.js MessagePack in Dapr Pub/Sub

```javascript
// msgpack-publisher.js
const { encode, decode } = require('@msgpack/msgpack');
const { DaprClient } = require('@dapr/dapr');

const client = new DaprClient();

async function publishProductEvent(product) {
  // Encode to MessagePack
  const encoded = encode(product);
  const base64 = Buffer.from(encoded).toString('base64');

  await client.pubsub.publish('pubsub', 'product-updates', {
    encoding: 'msgpack',
    data: base64,
    productId: product.id // keep key in JSON for routing
  });

  console.log(`Published product update: ${product.id}`);
}

// Subscriber decodes MessagePack
app.post('/events/product-updates', (req, res) => {
  const event = req.body;

  if (event.data.encoding === 'msgpack') {
    const bytes = Buffer.from(event.data.data, 'base64');
    const product = decode(bytes);
    console.log('Decoded product:', product);
  }

  res.sendStatus(200);
});
```

## When to Use MessagePack Over JSON

```bash
# Benchmark comparison for 1000 state saves
# JSON:        ~45ms encode, 78KB payload
# MessagePack: ~18ms encode, 42KB payload
# Protobuf:    ~12ms encode, 31KB payload

# MessagePack is ideal when:
# - You need binary efficiency without defining .proto schemas
# - Your team is polyglot and needs easy cross-language support
# - Payloads contain large numeric arrays (sensor data, ML features)
```

## Summary

MessagePack provides a practical middle ground between human-readable JSON and schema-heavy Protocol Buffers. In Dapr, you implement MessagePack by serializing manually before calling state save operations and deserializing after state reads. The approach works across all Dapr SDKs since it operates at the byte level, and it delivers meaningful payload size reductions for data-heavy microservices without requiring proto schema management.
