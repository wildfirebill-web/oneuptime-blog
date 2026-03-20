# How to Deploy Nuclio on Rancher - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Nuclio, Serverless, Kubernetes, Real-Time

Description: Guide to deploying Nuclio high-performance serverless framework on Rancher for real-time data processing.

## Introduction

Nuclio is a high-performance serverless framework designed for real-time data processing. It supports data bindings to Kafka, Kinesis, NATS, and more, making it ideal for ML inference pipelines and event-driven microservices.

## Step 1: Install Nuclio

```bash
# Add Nuclio Helm repository

helm repo add nuclio https://nuclio.github.io/nuclio/charts
helm repo update

# Create namespace
kubectl create namespace nuclio

# Install Nuclio
helm install nuclio nuclio/nuclio \
  --namespace nuclio \
  --set dashboard.enabled=true \
  --set registry.secretName=registry-credentials \
  --set controller.enabled=true \
  --version 0.14.0

# Wait for deployment
kubectl rollout status deployment nuclio-controller -n nuclio
kubectl rollout status deployment nuclio-dashboard -n nuclio
```

## Step 2: Configure Container Registry

```bash
# Create registry credentials secret
kubectl create secret docker-registry registry-credentials \
  --namespace nuclio \
  --docker-server=registry.example.com \
  --docker-username=your-username \
  --docker-password=your-password

# Configure Nuclio to use registry
kubectl set env deployment nuclio-controller \
  -n nuclio \
  NUCLIO_REGISTRY_URL=registry.example.com
```

## Step 3: Install nuctl CLI

```bash
# Download nuctl
curl -OL https://github.com/nuclio/nuclio/releases/download/1.12.0/nuctl-1.12.0-linux-amd64
chmod +x nuctl-1.12.0-linux-amd64
sudo mv nuctl-1.12.0-linux-amd64 /usr/local/bin/nuctl

nuctl version
```

## Step 4: Deploy a Python Function

```python
# process_event.py
import nuclio

def handler(context, event):
    context.logger.info_with('Processing event', 
                              body=event.body.decode('utf-8'))
    
    # Process the event
    result = f"Processed: {event.body.decode('utf-8')}"
    
    return nuclio.Response(
        headers={'Content-Type': 'application/text'},
        body=result,
        status_code=200
    )
```

```bash
# Deploy function
nuctl deploy process-event \
  --namespace nuclio \
  --path process_event.py \
  --runtime python:3.9 \
  --handler process_event:handler \
  --registry registry.example.com \
  --trigger-name http \
  --trigger-kind http \
  --replicas 2

# Test function
nuctl invoke process-event \
  --namespace nuclio \
  --method POST \
  --body "test data"
```

## Step 5: Function with Kafka Trigger

```yaml
# kafka-function.yaml
apiVersion: nuclio.io/v1beta1
kind: NuclioFunction
metadata:
  name: kafka-processor
  namespace: nuclio
spec:
  description: "Processes Kafka events"
  runtime: python:3.9
  handler: "kafka_handler:handler"
  
  image: registry.example.com/nuclio/kafka-processor:latest
  
  # Kafka trigger
  triggers:
    kafka-trigger:
      kind: kafka-cluster
      attributes:
        topics:
        - events-input
        brokers:
        - kafka.default.svc.cluster.local:9092
        consumerGroup: nuclio-processors
        initialOffset: latest
  
  # Resource configuration
  minReplicas: 1
  maxReplicas: 10
  
  resources:
    limits:
      cpu: "1"
      memory: "512Mi"
    requests:
      cpu: "100m"
      memory: "128Mi"
```

## Step 6: ML Inference Function

```python
# ml_inference.py
import nuclio
import numpy as np
import json

# Load model at startup (warm pod)
model = None

def init_context(context):
    global model
    # Load your ML model here
    context.logger.info("Loading model...")
    # model = load_model('/models/my_model')
    context.logger.info("Model loaded")

def handler(context, event):
    global model
    
    # Parse input
    input_data = json.loads(event.body)
    features = np.array(input_data['features'])
    
    # Run inference
    # prediction = model.predict(features)
    prediction = {"class": "cat", "confidence": 0.95}
    
    return nuclio.Response(
        body=json.dumps(prediction),
        headers={'Content-Type': 'application/json'},
        status_code=200
    )
```

## Step 7: HTTP Trigger with TLS

```yaml
# http-function.yaml
spec:
  triggers:
    http:
      kind: http
      maxWorkers: 8
      attributes:
        port: 8080
        ingresses:
          main:
            host: api.example.com
            paths:
            - /api/process
            tlsSecret: api-tls
```

## Monitoring Nuclio

```bash
# List all functions
nuctl get functions --namespace nuclio

# View function logs
nuctl logs kafka-processor --namespace nuclio --follow

# Access Nuclio dashboard
kubectl port-forward svc/nuclio-dashboard -n nuclio 8070:8070
# Open http://localhost:8070
```

## Conclusion

Nuclio excels at high-throughput, real-time data processing with its native integration with message queues and data streams. Its ability to load models and data at startup (rather than per-invocation) makes it particularly suitable for ML inference pipelines. Deploy Nuclio on Rancher when you need serverless processing at scale with rich data binding support.
