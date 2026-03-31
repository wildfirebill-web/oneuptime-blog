# How to Develop Dapr Pluggable Binding Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pluggable Component, Binding, gRPC, Extension

Description: Create custom Dapr input and output binding components using the pluggable component SDK to connect Dapr to external systems not in the built-in catalog.

---

## Dapr Bindings and Pluggability

Dapr bindings connect your application to external systems - both for receiving events (input bindings) and triggering external actions (output bindings). When no built-in binding exists for your system (a legacy mainframe, a proprietary IoT platform), pluggable bindings let you implement the interface yourself.

## Project Initialization

```bash
mkdir dapr-custom-binding && cd dapr-custom-binding
go mod init github.com/myorg/dapr-custom-binding
go get github.com/dapr-sandbox/components-go-sdk@latest
```

## Implementing the Output Binding

Output bindings respond to application-initiated operations:

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    dapr "github.com/dapr-sandbox/components-go-sdk"
    bindings "github.com/dapr-sandbox/components-go-sdk/bindings/v1"
    proto "github.com/dapr/dapr/pkg/proto/components/v1"
)

type WebhookBinding struct {
    webhookURL string
}

func (b *WebhookBinding) Init(ctx context.Context, req *proto.BindingInitRequest) (*proto.BindingInitResponse, error) {
    for _, m := range req.Metadata.Properties {
        if m.Key == "webhookURL" {
            b.webhookURL = m.Value
        }
    }
    log.Printf("Webhook binding initialized with URL: %s", b.webhookURL)
    return &proto.BindingInitResponse{}, nil
}

func (b *WebhookBinding) ListOperations(ctx context.Context, req *proto.ListOperationsRequest) (*proto.ListOperationsResponse, error) {
    return &proto.ListOperationsResponse{
        Operations: []string{"send", "delete"},
    }, nil
}

func (b *WebhookBinding) Invoke(ctx context.Context, req *proto.InvokeRequest) (*proto.InvokeResponse, error) {
    switch req.Operation {
    case "send":
        // Send to webhook
        payload := map[string]interface{}{
            "data": string(req.Data),
        }
        body, _ := json.Marshal(payload)
        log.Printf("Sending to webhook %s: %s", b.webhookURL, body)
        return &proto.InvokeResponse{Data: []byte(`{"status": "sent"}`)}, nil
    default:
        return nil, fmt.Errorf("unsupported operation: %s", req.Operation)
    }
}

func (b *WebhookBinding) Ping(ctx context.Context, req *proto.PingRequest) (*proto.PingResponse, error) {
    return &proto.PingResponse{}, nil
}
```

## Implementing the Input Binding

Input bindings deliver events to your application from external sources:

```go
type WebhookInputBinding struct {
    events chan *proto.ReadResponse
}

func (b *WebhookInputBinding) Read(req *proto.ReadRequest, stream proto.InputBinding_ReadServer) error {
    // Start an HTTP server to receive webhook events
    go b.startHTTPServer(stream)

    <-stream.Context().Done()
    return nil
}

func (b *WebhookInputBinding) startHTTPServer(stream proto.InputBinding_ReadServer) {
    // Accept incoming webhooks and forward to Dapr
    http.HandleFunc("/webhook", func(w http.ResponseWriter, r *http.Request) {
        body, _ := io.ReadAll(r.Body)
        stream.Send(&proto.ReadResponse{
            ContentType: "application/json",
            Data:        body,
        })
        w.WriteHeader(http.StatusOK)
    })
    http.ListenAndServe(":9000", nil)
}
```

## Registering Both Binding Types

```go
func main() {
    dapr.Register("custom-webhook",
        dapr.WithOutputBinding(func() bindings.OutputBinding {
            return &WebhookBinding{}
        }),
        dapr.WithInputBinding(func() bindings.InputBinding {
            return &WebhookInputBinding{
                events: make(chan *proto.ReadResponse, 100),
            }
        }),
    )

    dapr.MustRun()
}
```

## Component Manifest

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: webhook-binding
spec:
  type: bindings.custom-webhook
  version: v1
  metadata:
    - name: webhookURL
      value: "https://hooks.example.com/events"
    - name: direction
      value: "input, output"
```

## Using the Output Binding

```bash
# Invoke the output binding from your app
curl -X POST http://localhost:3500/v1.0/bindings/webhook-binding \
  -H "Content-Type: application/json" \
  -d '{"data": {"event": "order.created", "orderId": "456"}, "operation": "send"}'
```

## Summary

Dapr pluggable binding components enable bidirectional integration with any external system through a clean gRPC interface. Output bindings let your application trigger actions on external systems, while input bindings push external events into your app without polling. Combined with Dapr's built-in resiliency and tracing, pluggable bindings make legacy system integration straightforward.
