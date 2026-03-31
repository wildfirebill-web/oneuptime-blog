# How to Use Dapr Bindings as Serverless Triggers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, Serverless, Trigger, Event-Driven

Description: Use Dapr input bindings as serverless-style event triggers to activate your microservices from external events like cron schedules, webhooks, and cloud storage.

---

## Dapr Input Bindings as Serverless Triggers

Dapr input bindings eliminate the need for your services to poll external systems. Instead, the Dapr sidecar monitors the external source (a timer, a queue, a storage bucket) and calls your application's HTTP endpoint when an event arrives - a pattern identical to serverless function triggers but deployable on Kubernetes.

## Cron (Timer) Trigger

The most common serverless trigger - run code on a schedule:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: scheduled-job
spec:
  type: bindings.cron
  version: v1
  metadata:
    - name: schedule
      value: "@every 5m"    # Every 5 minutes
```

Application handler:

```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/scheduled-job', methods=['POST'])
def run_scheduled_job():
    print("Running scheduled job triggered by Dapr cron binding")
    # Do work here
    process_pending_orders()
    return '', 200

def process_pending_orders():
    import requests
    # Query state store for pending orders
    response = requests.get(
        'http://localhost:3500/v1.0/state/statestore/pending-orders'
    )
    orders = response.json()
    print(f"Processing {len(orders)} pending orders")
```

## AWS S3 Event Trigger

Trigger processing when files land in an S3 bucket:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: s3-trigger
spec:
  type: bindings.aws.s3
  version: v1
  metadata:
    - name: bucket
      value: "my-uploads-bucket"
    - name: region
      value: "us-east-1"
    - name: direction
      value: "input"
    - name: accessKey
      secretKeyRef:
        name: aws-secret
        key: accessKey
    - name: secretKey
      secretKeyRef:
        name: aws-secret
        key: secretKey
```

Handler:

```go
// Go - handle S3 upload trigger
func (h *Handler) HandleS3Event(w http.ResponseWriter, r *http.Request) {
    var event struct {
        FileName string `json:"fileName"`
        Bucket   string `json:"bucket"`
    }
    json.NewDecoder(r.Body).Decode(&event)

    log.Printf("New file uploaded: %s/%s", event.Bucket, event.FileName)
    // Process the uploaded file
    h.processFile(event.Bucket, event.FileName)

    w.WriteHeader(http.StatusOK)
}
```

## RabbitMQ Queue Trigger

Process messages from a RabbitMQ queue as a serverless trigger:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: work-queue
spec:
  type: bindings.rabbitmq
  version: v1
  metadata:
    - name: queueName
      value: "work-items"
    - name: host
      value: "amqp://rabbitmq:5672"
    - name: direction
      value: "input"
    - name: prefetchCount
      value: "10"
    - name: requeueInFailure
      value: "true"
```

## Kubernetes Event Trigger

Trigger on Kubernetes events:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: k8s-events
spec:
  type: bindings.kubernetes
  version: v1
  metadata:
    - name: namespace
      value: "production"
    - name: resyncPeriodInSec
      value: "10"
```

## HTTP Webhook Trigger

Receive external webhooks as binding events:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: github-webhook
spec:
  type: bindings.http
  version: v1
  metadata:
    - name: url
      value: "https://api.github.com/repos/myorg/myrepo"
    - name: direction
      value: "input"
```

## Comparing Trigger Approaches

```
| Trigger Type    | Dapr Binding Type      | Use Case                    |
|-----------------|------------------------|-----------------------------|
| Cron/Schedule   | bindings.cron          | Batch jobs, cleanup tasks   |
| Storage Upload  | bindings.aws.s3        | File processing pipelines   |
| Message Queue   | bindings.rabbitmq      | Work queue consumers        |
| HTTP Webhook    | bindings.http          | External event integration  |
| Kubernetes      | bindings.kubernetes    | Cluster event reactions     |
```

## Summary

Dapr input bindings transform external event sources into HTTP calls on your application, enabling a serverless-trigger pattern without serverless infrastructure. By declaratively defining bindings for cron schedules, cloud storage events, or message queues, you get event-driven activation with Kubernetes reliability, Dapr resiliency policies, and the ability to respond to dozens of event sources with a simple HTTP endpoint in your application.
