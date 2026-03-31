# How to Develop Dapr Pluggable Pub/Sub Components

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pluggable Component, Pub/Sub, gRPC, Extension

Description: Build a custom Dapr pluggable pub/sub component using the Go component SDK to integrate any message broker with the Dapr pub/sub building block.

---

## Why Build a Pluggable Pub/Sub Component?

While Dapr supports 20+ pub/sub brokers out of the box, you may need to integrate with a proprietary messaging system, an internal event bus, or a broker not yet in the Dapr catalog. Pluggable pub/sub components implement the Dapr pub/sub gRPC interface and run as sidecar containers.

## Project Setup

```bash
mkdir dapr-custom-pubsub && cd dapr-custom-pubsub
go mod init github.com/myorg/dapr-custom-pubsub

go get github.com/dapr-sandbox/components-go-sdk@latest
```

## Implementing the Pub/Sub Interface

The pub/sub interface requires Init, Features, Publish, Subscribe, and Ping:

```go
package main

import (
    "context"
    "sync"

    dapr "github.com/dapr-sandbox/components-go-sdk"
    pubsub "github.com/dapr-sandbox/components-go-sdk/pubsub/v1"
    proto "github.com/dapr/dapr/pkg/proto/components/v1"
)

type InMemoryPubSub struct {
    mu          sync.RWMutex
    subscribers map[string][]chan *proto.TopicEventRequest
}

func (p *InMemoryPubSub) Init(ctx context.Context, req *proto.PubSubInitRequest) (*proto.PubSubInitResponse, error) {
    p.subscribers = make(map[string][]chan *proto.TopicEventRequest)
    return &proto.PubSubInitResponse{}, nil
}

func (p *InMemoryPubSub) Features(ctx context.Context, req *proto.FeaturesRequest) (*proto.FeaturesResponse, error) {
    return &proto.FeaturesResponse{
        Features: []string{"MESSAGE_TTL"},
    }, nil
}

func (p *InMemoryPubSub) Publish(ctx context.Context, req *proto.PublishRequest) (*proto.PublishResponse, error) {
    p.mu.RLock()
    defer p.mu.RUnlock()

    topic := req.Topic
    if subs, ok := p.subscribers[topic]; ok {
        event := &proto.TopicEventRequest{
            Data:        req.Data,
            DataContentType: req.DataContentType,
            Topic:       topic,
            PubsubName:  req.PubsubName,
        }
        for _, ch := range subs {
            select {
            case ch <- event:
            default:
                // Non-blocking send
            }
        }
    }
    return &proto.PublishResponse{}, nil
}

func (p *InMemoryPubSub) Subscribe(req *proto.SubscribeRequest, stream proto.PubSub_SubscribeServer) error {
    p.mu.Lock()
    ch := make(chan *proto.TopicEventRequest, 100)
    p.subscribers[req.Topic] = append(p.subscribers[req.Topic], ch)
    p.mu.Unlock()

    for {
        select {
        case event := <-ch:
            if err := stream.Send(event); err != nil {
                return err
            }
        case <-stream.Context().Done():
            return nil
        }
    }
}
```

## Main Entry Point

```go
func main() {
    store := &InMemoryPubSub{}

    dapr.Register("custom-pubsub", dapr.WithPubSub(func() pubsub.PubSub {
        return store
    }))

    dapr.MustRun()
}
```

## Component Manifest

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: custom-pubsub
spec:
  type: pubsub.custom-pubsub
  version: v1
  metadata:
    - name: brokerURL
      value: "internal-broker:8080"
    - name: maxRetries
      value: "3"
```

## Subscription Configuration

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-subscription
spec:
  pubsubname: custom-pubsub
  topic: orders
  routes:
    default: /orders/handler
```

## Testing the Component

```bash
# Build and run
go build -o custom-pubsub .
DAPR_COMPONENT_SOCKET_FOLDER=/tmp/dapr-components ./custom-pubsub &

# Test publish
curl -X POST http://localhost:3500/v1.0/publish/custom-pubsub/orders \
  -H "Content-Type: application/json" \
  -d '{"orderId": "123", "item": "widget"}'
```

## Acknowledgment Handling

Properly handle message acknowledgments to prevent redelivery:

```go
func (p *InMemoryPubSub) Subscribe(req *proto.SubscribeRequest, stream proto.PubSub_SubscribeServer) error {
    for event := range p.getEvents(req.Topic) {
        if err := stream.Send(event); err != nil {
            return err
        }
        // Wait for ack/nack before processing next message
        status, err := stream.Recv()
        if err != nil {
            return err
        }
        if status.Status == proto.TopicEventResponse_RETRY {
            // Re-enqueue for retry
            p.requeue(event)
        }
    }
    return nil
}
```

## Summary

Dapr pluggable pub/sub components extend the Dapr messaging model to any broker via a gRPC streaming interface. By implementing Publish and Subscribe with proper acknowledgment handling, you get the full Dapr pub/sub feature set - routing, deadletter queues, and resiliency policies - on top of your custom messaging infrastructure.
