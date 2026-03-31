# How to Use Dapr Service Invocation with Different Content Types

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Content Type, HTTP, Service Invocation, Serialization

Description: Learn how to use Dapr service invocation with JSON, XML, Protobuf, form data, and binary content types, including proper header configuration and encoding.

---

## Content Type Handling in Dapr

Dapr forwards the `Content-Type` header to the target service unchanged. The sidecar does not parse or transform the request body. This means any content type your service supports can be used.

## JSON (Most Common)

```bash
curl -X POST http://localhost:3500/v1.0/invoke/order-service/method/orders \
  -H "Content-Type: application/json" \
  -d '{"item": "widget", "qty": 5}'
```

## XML

```bash
curl -X POST http://localhost:3500/v1.0/invoke/order-service/method/orders \
  -H "Content-Type: application/xml" \
  -d '<order><item>widget</item><qty>5</qty></order>'
```

```javascript
// Server side - parse XML
const xml2js = require('xml2js');
app.post('/orders', async (req, res) => {
  if (req.is('application/xml')) {
    const parsed = await xml2js.parseStringPromise(req.body);
    // use parsed.order.item[0]
  }
});
```

## Form Data

```bash
curl -X POST http://localhost:3500/v1.0/invoke/order-service/method/orders \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "item=widget&qty=5"
```

## Multipart Form Data (File Upload)

```bash
curl -X POST http://localhost:3500/v1.0/invoke/document-service/method/upload \
  -F "file=@/path/to/document.pdf" \
  -F "description=Annual Report"
```

## Protocol Buffers (Protobuf)

For binary Protobuf payloads over HTTP:

```bash
# Encode protobuf message and send
protoc --encode=orders.OrderRequest orders.proto < order.txt | \
  curl -X POST http://localhost:3500/v1.0/invoke/order-service/method/orders \
    -H "Content-Type: application/protobuf" \
    --data-binary @-
```

In Node.js:

```javascript
const protobuf = require('protobufjs');
const root = await protobuf.load('orders.proto');
const OrderRequest = root.lookupType('orders.OrderRequest');
const message = OrderRequest.create({ item: 'widget', qty: 5 });
const buffer = OrderRequest.encode(message).finish();

await axios.post(
  'http://localhost:3500/v1.0/invoke/order-service/method/orders',
  buffer,
  { headers: { 'Content-Type': 'application/protobuf' } }
);
```

## Content Negotiation with Accept Header

```bash
# Request JSON response even when calling a service that supports multiple formats
curl http://localhost:3500/v1.0/invoke/order-service/method/orders/123 \
  -H "Accept: application/json"
```

## NDJSON (Newline-Delimited JSON)

```bash
curl -X POST http://localhost:3500/v1.0/invoke/batch-service/method/process \
  -H "Content-Type: application/x-ndjson" \
  --data-binary $'{"id":1}\n{"id":2}\n{"id":3}\n'
```

## Summary

Dapr service invocation passes the `Content-Type` header and request body through to the target service without modification. Use JSON, XML, form data, multipart, Protobuf, or any other content type your service supports. Set the correct `Content-Type` header on the request and implement the corresponding parser in your service handler.
