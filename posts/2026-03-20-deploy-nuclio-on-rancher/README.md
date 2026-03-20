# How to Deploy Nuclio on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Nuclio, Serverless, Real-Time Processing, Kubernetes, GPU

Description: Deploy Nuclio serverless platform on Rancher for real-time data processing with GPU support, multiple trigger types, and the Nuclio dashboard.

## Introduction

Nuclio is a high-performance serverless framework optimized for real-time event-driven processing. Unlike general-purpose serverless frameworks, Nuclio is designed for data science workloads, GPU acceleration, and high-throughput event processing. It supports Python, Go, Java, Node.js, and .NET.

## Step 1: Install Nuclio via Helm

```bash
helm repo add nuclio https://nuclio.github.io/nuclio/charts
helm repo update

helm install nuclio nuclio/nuclio \
  --namespace nuclio \
  --create-namespace \
  --set dashboard.enabled=true \
  --set dashboard.serviceType=NodePort

kubectl get pods -n nuclio
```

## Step 2: Access the Nuclio Dashboard

```bash
# Get the dashboard port

DASHBOARD_PORT=$(kubectl get svc nuclio-dashboard -n nuclio \
  -o jsonpath='{.spec.ports[0].nodePort}')

# Access the dashboard
echo "Dashboard: http://<node-ip>:$DASHBOARD_PORT"
```

## Step 3: Deploy a Function via Dashboard

1. Open the Nuclio Dashboard
2. Click **New Function**
3. Choose runtime (Python 3.8)
4. Enter function code:

```python
# Python function handler
def handler(context, event):
    context.logger.info(f"Processing event: {event.body}")

    # Process the event
    result = {
        "processed": True,
        "input": event.body.decode('utf-8'),
        "timestamp": event.timestamp.isoformat()
    }

    return context.Response(
        body=str(result),
        status_code=200,
        content_type="application/json"
    )
```

## Step 4: Deploy via nuctl CLI

```bash
# Install nuctl
curl -s https://api.github.com/repos/nuclio/nuclio/releases/latest | \
  grep browser_download_url | grep linux-amd64 | grep nuctl | \
  cut -d '"' -f 4 | xargs curl -L -o nuctl && chmod +x nuctl

# Deploy a function
nuctl deploy my-function \
  --namespace nuclio \
  --path /path/to/function \
  --runtime python:3.8 \
  --handler handler:handler \
  --triggers '{"http": {"kind": "http", "attributes": {"port": 8080}}}' \
  --platform kube
```

## Step 5: Configure Kafka Trigger

```yaml
# function-config.yaml
spec:
  triggers:
    kafka-trigger:
      kind: kafka-cluster
      attributes:
        brokers:
          - kafka.messaging.svc.cluster.local:9092
        topics:
          - events
        consumerGroup: nuclio-consumer
        maxBatchSize: 100
```

## Step 6: Configure GPU Support

```yaml
# For ML inference functions requiring GPU
spec:
  resources:
    limits:
      nvidia.com/gpu: "1"    # Request one GPU
  platform:
    attributes:
      restartPolicy:
        name: always
```

## Conclusion

Nuclio on Rancher excels at high-throughput real-time processing workloads. Its GPU support makes it the natural choice for ML inference serverless functions. The built-in dashboard provides an accessible interface for data scientists who may not be Kubernetes experts, while the CLI supports GitOps-friendly deployments.
