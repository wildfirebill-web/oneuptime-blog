# How to Use Dapr Service Invocation with REST APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, REST, HTTP, Service Invocation, API

Description: Learn how to expose and consume RESTful APIs between Dapr-enabled microservices, using proper HTTP methods, status codes, and content negotiation.

---

## REST API Conventions with Dapr

Dapr service invocation is designed to work naturally with REST APIs. Your application exposes its REST endpoints normally, and callers invoke them through the Dapr sidecar using the service app ID as the host.

## Exposing a REST API in Your Service

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// Standard REST endpoints
app.get('/orders', async (req, res) => {
  const orders = await db.getOrders();
  res.json(orders);
});

app.get('/orders/:id', async (req, res) => {
  const order = await db.getOrder(req.params.id);
  if (!order) return res.status(404).json({ error: 'Order not found' });
  res.json(order);
});

app.post('/orders', async (req, res) => {
  const order = await db.createOrder(req.body);
  res.status(201).json(order);
});

app.put('/orders/:id', async (req, res) => {
  const order = await db.updateOrder(req.params.id, req.body);
  res.json(order);
});

app.delete('/orders/:id', async (req, res) => {
  await db.deleteOrder(req.params.id);
  res.status(204).send();
});

app.listen(3000);
```

## Calling REST Endpoints via Dapr

```bash
# GET collection
curl http://localhost:3500/v1.0/invoke/order-service/method/orders

# GET single resource
curl http://localhost:3500/v1.0/invoke/order-service/method/orders/123

# POST to create
curl -X POST http://localhost:3500/v1.0/invoke/order-service/method/orders \
  -H "Content-Type: application/json" \
  -d '{"item": "widget", "qty": 5}'

# PUT to update
curl -X PUT http://localhost:3500/v1.0/invoke/order-service/method/orders/123 \
  -H "Content-Type: application/json" \
  -d '{"qty": 10}'

# DELETE
curl -X DELETE http://localhost:3500/v1.0/invoke/order-service/method/orders/123
```

## Content Negotiation

```bash
# Request JSON
curl http://localhost:3500/v1.0/invoke/order-service/method/orders \
  -H "Accept: application/json"

# Request specific version
curl http://localhost:3500/v1.0/invoke/order-service/method/v2/orders \
  -H "Accept: application/vnd.orders.v2+json"
```

## Handling REST Error Responses

Dapr propagates HTTP status codes from the target service. Handle them appropriately:

```javascript
try {
  const res = await axios.delete(
    'http://localhost:3500/v1.0/invoke/order-service/method/orders/123'
  );
} catch (err) {
  switch (err.response?.status) {
    case 404: console.error('Order not found'); break;
    case 409: console.error('Conflict - order already deleted'); break;
    case 422: console.error('Validation error', err.response.data); break;
  }
}
```

## Query Parameters in REST Calls

```bash
curl "http://localhost:3500/v1.0/invoke/order-service/method/orders?status=pending&limit=20&offset=0"
```

## Summary

Dapr service invocation works seamlessly with RESTful APIs. Expose standard REST endpoints in your service and invoke them through the Dapr sidecar by prepending `http://localhost:3500/v1.0/invoke/{app-id}/method/` to the path. All HTTP methods, status codes, query parameters, and content negotiation headers are preserved.
