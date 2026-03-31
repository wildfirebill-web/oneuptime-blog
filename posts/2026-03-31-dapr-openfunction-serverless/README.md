# How to Use Dapr with OpenFunction Serverless Platform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, OpenFunction, Serverless, Function, Kubernetes, Event-Driven

Description: Deploy event-driven serverless functions on Kubernetes using OpenFunction with Dapr bindings and pub/sub as event sources and sinks for your function triggers.

---

## What Is OpenFunction?

OpenFunction is a cloud-agnostic, open-source serverless platform for Kubernetes that uses Dapr as its event framework. Functions in OpenFunction can subscribe to Dapr pub/sub topics or Dapr input bindings as event triggers, and publish to output bindings as sinks.

## Architecture

```text
Event Source (Kafka/Redis/HTTP)
        |
    Dapr Input Binding
        |
    OpenFunction Runtime
        |
    Your Function Code
        |
    Dapr Output Binding / Pub/Sub
        |
   Event Sink (Database/Queue/Service)
```

## Installing OpenFunction

```bash
# Add Helm repo
helm repo add openfunction https://openfunction.github.io/charts/
helm repo update

# Install with Dapr integration
helm install openfunction openfunction/openfunction \
  --namespace openfunction \
  --create-namespace \
  --set global.Dapr.enabled=true \
  --wait
```

Verify Dapr is integrated:

```bash
kubectl get pods -n openfunction
# dapr-operator, dapr-sentry, openfunction-controller
```

## Writing an OpenFunction Function

Write a function in Go that processes Dapr pub/sub events:

```go
package main

import (
    "context"
    "fmt"
    ofctx "github.com/OpenFunction/functions-framework-go/context"
)

func ProcessOrder(ctx ofctx.Context, in []byte) (ofctx.Out, error) {
    fmt.Printf("Received order event: %s\n", string(in))

    // Publish result to output binding
    ctx.Send(ofctx.NewUserData().
        WithData([]byte(`{"status": "processed"}`)).
        WithPort("processed-orders"))

    return ctx.ReturnOnSuccess(), nil
}
```

## Function Definition YAML

Create an OpenFunction Function resource:

```yaml
apiVersion: core.openfunction.io/v1beta2
kind: Function
metadata:
  name: order-processor
  namespace: production
spec:
  version: v2.0.0
  image: myregistry/order-processor:latest
  imageCredentials:
    name: push-secret
  build:
    builder: openfunction/builder-go:latest
    env:
      FUNC_NAME: ProcessOrder
    srcRepo:
      url: https://github.com/myorg/order-processor.git
      sourceSubPath: functions/order-processor
  serving:
    template:
      annotations:
        dapr.io/log-level: "info"
    runtime: async
    scaleOptions:
      minReplicas: 0
      maxReplicas: 10
    triggers:
      dapr:
        - name: kafka-pubsub
          topic: orders
    outputs:
      - dapr:
          name: results-pubsub
          topic: processed-orders
          operation: publish
    bindings:
      kafka-pubsub:
        type: bindings.kafka
        version: v1
        metadata:
          - name: brokers
            value: kafka:9092
          - name: topics
            value: orders
          - name: consumerGroup
            value: order-processor
```

## Scaling with KEDA

OpenFunction integrates with KEDA for event-driven scaling of Dapr-triggered functions:

```yaml
scaleOptions:
  keda:
    scaledObject:
      pollingInterval: 15
      cooldownPeriod: 300
      triggers:
        - type: kafka
          metadata:
            bootstrapServers: kafka:9092
            consumerGroup: order-processor
            topic: orders
            lagThreshold: "100"   # Scale up when lag > 100 messages
  minReplicas: 0
  maxReplicas: 20
```

## Monitoring OpenFunction with Dapr Metrics

Both OpenFunction and Dapr expose Prometheus metrics:

```bash
# Function invocation count
openfunction_function_invocation_total

# Dapr pub/sub message processing rate (from the function's sidecar)
dapr_pubsub_incoming_messages_total{app_id="order-processor"}

# Scale up events from KEDA
keda_scaler_active
```

## Local Development with Dapr

Test the function locally using Dapr standalone mode:

```bash
dapr run \
  --app-id order-processor \
  --app-port 8080 \
  --resources-path ./components \
  -- go run ./functions/order-processor/main.go

# Publish a test event
dapr publish \
  --publish-app-id order-processor \
  --pubsub kafka-pubsub \
  --topic orders \
  --data '{"orderId": "test-123"}'
```

## Summary

OpenFunction integrates Dapr as its event framework, enabling serverless functions that scale to zero on Kubernetes while consuming Dapr pub/sub topics and bindings as event triggers. KEDA provides event-driven autoscaling based on message lag. The function code uses a simple handler interface, with all Dapr configuration declared in the Function YAML, making event-driven serverless development straightforward on any Kubernetes cluster.
