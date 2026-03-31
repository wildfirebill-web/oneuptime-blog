# How to Handle Large Payloads in Dapr Service Invocation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Performance, Payload, Service Invocation, Streaming

Description: Learn how to handle large payloads in Dapr service invocation by tuning sidecar limits, using chunked transfers, and offloading large data to object storage.

---

## Default Payload Limits

Dapr's HTTP server has a default maximum request body size of 4MB. Payloads exceeding this return a 413 error. For most microservice use cases, this is sufficient, but large file transfers or batch operations may exceed it.

## Increasing the Max Request Body Size

Configure the sidecar to accept larger payloads:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "document-service"
  dapr.io/http-max-request-size: "64"  # MB
```

Or for self-hosted:

```bash
dapr run --app-id document-service --http-max-request-size 64 -- node app.js
```

## Increasing Read Buffer Size

For large headers alongside large bodies:

```yaml
annotations:
  dapr.io/http-read-buffer-size: "16"  # KB, default is 4KB
```

## Using Chunked Transfer Encoding

For streaming large responses, use chunked transfer encoding in your service:

```javascript
app.get('/large-report', (req, res) => {
  res.setHeader('Transfer-Encoding', 'chunked');
  res.setHeader('Content-Type', 'application/json');

  res.write('[');
  for (let i = 0; i < 10000; i++) {
    res.write(JSON.stringify({ index: i, value: `item-${i}` }));
    if (i < 9999) res.write(',');
  }
  res.write(']');
  res.end();
});
```

## Pattern: Claim Check (Preferred Approach)

For very large payloads, use the Claim Check pattern - store the data in object storage and pass only a reference:

```javascript
// Sender
const { BlobServiceClient } = require('@azure/storage-blob');
const client = BlobServiceClient.fromConnectionString(connStr);

async function invokeWithLargePayload(data) {
  // Upload to blob storage
  const blobId = `payload-${Date.now()}.json`;
  const container = client.getContainerClient('payloads');
  await container.uploadBlockBlob(blobId, JSON.stringify(data), Buffer.byteLength(JSON.stringify(data)));

  // Send only the reference
  return await axios.post(
    'http://localhost:3500/v1.0/invoke/processor-service/method/process',
    { payloadRef: blobId },
    { headers: { 'Content-Type': 'application/json' } }
  );
}
```

```javascript
// Receiver
app.post('/process', async (req, res) => {
  const { payloadRef } = req.body;
  const container = client.getContainerClient('payloads');
  const blob = container.getBlobClient(payloadRef);
  const data = JSON.parse((await blob.downloadToBuffer()).toString());
  // process data
  res.json({ status: 'processed' });
});
```

## Dapr Binding for Large File Transfers

For large binary files, use Dapr bindings instead of service invocation:

```bash
# Output binding to write file
curl -X POST http://localhost:3500/v1.0/bindings/local-storage \
  -H "Content-Type: application/json" \
  -d '{"operation": "create", "metadata": {"fileName": "report.pdf"}, "data": "<base64-encoded-content>"}'
```

## Summary

Handle large payloads in Dapr service invocation by increasing the `http-max-request-size` annotation, using chunked transfer encoding for streaming responses, or preferring the Claim Check pattern where large data is stored in object storage and only a reference is passed. For binary files, Dapr bindings are more appropriate than service invocation.
