# How to Use Dapr with Knative Serving

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Knative, Serverless, Scale To Zero, Kubernetes, Event-Driven

Description: Run Dapr-enabled services as Knative Serving workloads to get HTTP-triggered scale-to-zero capabilities while retaining full access to Dapr building blocks.

---

## Why Combine Dapr with Knative Serving

Knative Serving provides HTTP-based autoscaling and scale-to-zero for containerized services. Dapr provides state management, pub/sub, and secrets. Together they enable stateless serverless functions with access to Dapr's infrastructure abstractions.

Use cases:
- Webhook handlers that process Dapr pub/sub events via HTTP
- REST APIs that use Dapr state and secrets with scale-to-zero
- Event-driven processing pipelines

## Installing Knative and Dapr

```bash
# Install Knative Serving
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-core.yaml

# Install Knative Kourier networking layer
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.13.0/kourier.yaml

# Install Dapr
dapr init -k
```

## Creating a Dapr-Enabled Knative Service

Knative Services use pod templates. Add Dapr annotations to the pod template:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: order-handler
  namespace: production
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-handler"
        dapr.io/app-port: "8080"
        dapr.io/log-level: "info"
        autoscaling.knative.dev/target: "100"      # Requests per replica
        autoscaling.knative.dev/scale-to-zero-pod-retention-period: "60s"
    spec:
      containers:
        - image: myregistry/order-handler:latest
          ports:
            - containerPort: 8080
          env:
            - name: DAPR_HTTP_PORT
              value: "3500"
```

## Handling Dapr Sidecar with Knative Scale-to-Zero

A challenge with Dapr and scale-to-zero is that both the app container and the Dapr sidecar must terminate gracefully. Configure shutdown delay:

```yaml
annotations:
  dapr.io/enabled: "true"
  dapr.io/app-id: "order-handler"
  dapr.io/graceful-shutdown-seconds: "30"
```

Also configure Knative's drain timeout:

```yaml
annotations:
  autoscaling.knative.dev/initial-scale: "0"
  autoscaling.knative.dev/scale-to-zero-grace-period: "30s"
```

## Application Code

The service receives HTTP requests and uses Dapr for state and secrets:

```go
package main

import (
    "encoding/json"
    "net/http"
    dapr "github.com/dapr/go-sdk/client"
)

func handleOrder(w http.ResponseWriter, r *http.Request) {
    client, _ := dapr.NewClient()
    defer client.Close()

    var order map[string]interface{}
    json.NewDecoder(r.Body).Decode(&order)

    // Save order state via Dapr
    data, _ := json.Marshal(order)
    client.SaveState(r.Context(), "statestore",
        order["id"].(string), data, nil)

    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"status":"accepted"}`))
}

func main() {
    http.HandleFunc("/orders", handleOrder)
    http.ListenAndServe(":8080", nil)
}
```

## Triggering from Dapr Pub/Sub

Configure a Dapr subscription that delivers events to the Knative Service URL:

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-subscription
spec:
  pubsubname: pubsub
  topic: orders
  routes:
    default: /orders     # HTTP POST to the Knative Service
  scopes:
    - order-handler
```

Knative will scale from zero when events arrive.

## Monitoring Scale Events

Watch scaling activity:

```bash
# Watch pod count change
kubectl get pods -n production -l serving.knative.dev/service=order-handler -w

# Check Knative autoscaler metrics
kubectl get hpa -n production

# Knative metrics in Prometheus
knative_serving_autoscaler_desired_pods{revision="order-handler-v1"}
```

## Traffic Splitting for Canary

Knative supports traffic splitting between revisions natively:

```yaml
spec:
  traffic:
    - revisionName: order-handler-v1
      percent: 90
    - revisionName: order-handler-v2
      percent: 10
```

Both revisions share the same Dapr components (statestore, pubsub).

## Summary

Knative Serving and Dapr complement each other well: Knative provides HTTP-based scale-to-zero autoscaling while Dapr provides state, pub/sub, and secrets. Configuring graceful shutdown annotations for both the Dapr sidecar and Knative's drain timeout ensures clean scale-to-zero transitions. Dapr subscriptions can trigger Knative Services when pub/sub events arrive, creating a fully event-driven serverless architecture.
