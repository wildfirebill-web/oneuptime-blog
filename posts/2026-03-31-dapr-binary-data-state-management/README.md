# How to Handle Binary Data in Dapr State Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, State Management, Binary, Encoding, Storage

Description: Store and retrieve binary data in Dapr state stores using base64 encoding, byte arrays, and content type metadata across .NET, Go, and Python SDK examples.

---

## Binary Data Challenges in Dapr State

Dapr's state management API communicates over HTTP/gRPC using JSON envelopes. Binary data (images, documents, serialized objects) cannot be directly embedded in JSON, so it must be encoded. The two approaches are: base64 encoding in JSON payloads, or using the Dapr gRPC API with byte arrays.

## .NET: Storing Binary Files

```csharp
using Dapr.Client;

public class BinaryStateService
{
    private readonly DaprClient _dapr;
    private const string Store = "statestore";

    public BinaryStateService(DaprClient dapr)
    {
        _dapr = dapr;
    }

    // Store raw bytes (e.g., an image or PDF)
    public async Task SaveBinaryAsync(string key, byte[] data, string contentType)
    {
        var wrapper = new BinaryStateWrapper
        {
            ContentType = contentType,
            Data = Convert.ToBase64String(data),
            Size = data.Length,
            StoredAt = DateTime.UtcNow
        };

        await _dapr.SaveStateAsync(Store, key, wrapper);
    }

    public async Task<byte[]?> GetBinaryAsync(string key)
    {
        var wrapper = await _dapr.GetStateAsync<BinaryStateWrapper>(Store, key);
        if (wrapper == null) return null;

        return Convert.FromBase64String(wrapper.Data);
    }

    // Store a file from disk
    public async Task SaveFileAsync(string key, string filePath)
    {
        var bytes = await File.ReadAllBytesAsync(filePath);
        var contentType = GetContentType(filePath);
        await SaveBinaryAsync(key, bytes, contentType);
    }

    private string GetContentType(string filePath)
    {
        return Path.GetExtension(filePath).ToLower() switch
        {
            ".jpg" or ".jpeg" => "image/jpeg",
            ".png" => "image/png",
            ".pdf" => "application/pdf",
            _ => "application/octet-stream"
        };
    }
}

public class BinaryStateWrapper
{
    public string ContentType { get; set; } = "";
    public string Data { get; set; } = ""; // base64 encoded
    public long Size { get; set; }
    public DateTime StoredAt { get; set; }
}
```

## Go: Binary State with Chunking

```go
// binary_state.go
package main

import (
    "context"
    "encoding/base64"
    "encoding/json"
    "fmt"

    dapr "github.com/dapr/go-sdk/client"
)

const maxChunkSize = 1024 * 1024 // 1MB chunks

type BinaryChunk struct {
    ChunkIndex int    `json:"chunkIndex"`
    TotalChunks int  `json:"totalChunks"`
    Data       string `json:"data"` // base64
    Key        string `json:"key"`
}

type BinaryManifest struct {
    TotalChunks int    `json:"totalChunks"`
    TotalSize   int    `json:"totalSize"`
    ContentType string `json:"contentType"`
}

func saveLargeBinary(ctx context.Context, client dapr.Client,
    key string, data []byte, contentType string) error {

    chunks := splitIntoChunks(data, maxChunkSize)
    manifest := BinaryManifest{
        TotalChunks: len(chunks),
        TotalSize:   len(data),
        ContentType: contentType,
    }

    // Save manifest
    manifestBytes, _ := json.Marshal(manifest)
    if err := client.SaveState(ctx, "statestore",
        "manifest-"+key, manifestBytes, nil); err != nil {
        return fmt.Errorf("failed to save manifest: %w", err)
    }

    // Save each chunk
    for i, chunk := range chunks {
        chunkData := BinaryChunk{
            ChunkIndex:  i,
            TotalChunks: len(chunks),
            Data:        base64.StdEncoding.EncodeToString(chunk),
            Key:         key,
        }
        chunkBytes, _ := json.Marshal(chunkData)
        chunkKey := fmt.Sprintf("chunk-%s-%d", key, i)

        if err := client.SaveState(ctx, "statestore",
            chunkKey, chunkBytes, nil); err != nil {
            return fmt.Errorf("failed to save chunk %d: %w", i, err)
        }
    }
    return nil
}

func splitIntoChunks(data []byte, size int) [][]byte {
    var chunks [][]byte
    for len(data) > 0 {
        if len(data) < size {
            size = len(data)
        }
        chunks = append(chunks, data[:size])
        data = data[size:]
    }
    return chunks
}
```

## Python: Binary State with Metadata

```python
# binary_state.py
import base64
import hashlib
import json
from dapr.clients import DaprClient

def save_binary_state(key: str, data: bytes, content_type: str = "application/octet-stream"):
    """Save binary data to Dapr state with metadata."""
    checksum = hashlib.md5(data).hexdigest()

    payload = json.dumps({
        "content_type": content_type,
        "data": base64.b64encode(data).decode('ascii'),
        "size": len(data),
        "checksum": checksum
    })

    with DaprClient() as client:
        client.save_state(
            store_name="statestore",
            key=key,
            value=payload,
            state_metadata={"contentType": "application/json"}
        )
    return checksum

def get_binary_state(key: str) -> tuple[bytes, str]:
    """Retrieve binary data from Dapr state, returns (data, content_type)."""
    with DaprClient() as client:
        result = client.get_state(store_name="statestore", key=key)

    payload = json.loads(result.data)
    data = base64.b64decode(payload["data"])

    # Verify integrity
    checksum = hashlib.md5(data).hexdigest()
    if checksum != payload["checksum"]:
        raise ValueError(f"Data integrity check failed for key: {key}")

    return data, payload["content_type"]
```

## Summary

Dapr state management handles binary data through base64 encoding wrapped in JSON payloads. For large binaries, implement chunking to stay within state store size limits (typically 2-16MB per key). Always include metadata like content type and checksums to enable proper reconstruction and integrity verification. For extremely large files, consider storing only a reference key in Dapr state and using object storage (S3, Azure Blob) for the actual binary content.
