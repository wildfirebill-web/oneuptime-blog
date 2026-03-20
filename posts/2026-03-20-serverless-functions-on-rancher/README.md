# How to Deploy Serverless Functions on Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Serverless, Functions, Kubernetes, OpenFaaS, Knative

Description: Deploy serverless functions on Rancher with a practical comparison of frameworks, function lifecycle management, and production deployment patterns.

## Introduction

Serverless functions on Kubernetes abstract away pod management, routing, and scaling. This guide covers practical function deployment using OpenFaaS, a comparison of available frameworks, and patterns for production-grade function management in Rancher.

## Framework Comparison

| Framework | Cold Start | Scale-to-Zero | Language Support | Best For |
|---|---|---|---|---|
| Knative | Medium | Yes | Any | HTTP workloads |
| OpenFaaS | Fast | Yes | Any | General purpose |
| Fission | Fastest | Yes | 8+ languages | Latency-sensitive |
| KEDA Jobs | N/A | Yes | Any | Batch processing |

## Step 1: Prepare Your Function

Write a function with a clear contract:

```python
# process_image.py

import json
import base64
from PIL import Image
import io

def handle(event, context):
    """Process an image and return metadata."""
    try:
        # Decode base64 image from request body
        image_data = base64.b64decode(event.body)
        img = Image.open(io.BytesIO(image_data))

        result = {
            "width": img.width,
            "height": img.height,
            "format": img.format,
            "mode": img.mode
        }

        return json.dumps(result), 200

    except Exception as e:
        return json.dumps({"error": str(e)}), 500
```

## Step 2: Package as a Dockerfile

```dockerfile
# Use the OpenFaaS Python template as a base
FROM ghcr.io/openfaas/classic-watchdog:0.2.1 as watchdog

FROM python:3.11-slim
COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

WORKDIR /home/app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY handler.py .

ENV fprocess="python handler.py"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:5000"

HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1
CMD ["fwatchdog"]
```

## Step 3: Deploy and Configure Routing

```bash
# Deploy with OpenFaaS
faas-cli deploy \
  --image myregistry/process-image:latest \
  --name process-image \
  --gateway http://openfaas-gateway:8080 \
  --memory-limit 256m \
  --cpu-limit 250m \
  --env MAX_INFLIGHT=10 \
  --label com.openfaas.scale.min=0 \
  --label com.openfaas.scale.max=10 \
  --label com.openfaas.scale.factor=20
```

## Step 4: Implement Function Versioning

```bash
# Deploy with version tag for blue/green testing
faas-cli deploy \
  --image myregistry/process-image:v2.0 \
  --name process-image-v2 \
  --gateway http://openfaas-gateway:8080

# Test v2 before promoting
curl -X POST http://openfaas-gateway:8080/function/process-image-v2 \
  --data-binary @test-image.jpg

# Promote v2 to production by updating the canary deployment
```

## Step 5: Configure Timeouts and Retries

```yaml
# Function configuration with production settings
annotations:
  com.openfaas.timeout: "30s"           # Function execution timeout
  com.openfaas.hard-timeout: "35s"      # Hard kill timeout
  topic: "image-processor"              # Subscribe to NATS/Kafka topic for async

environment:
  - MAX_INFLIGHT: "10"                  # Concurrent requests per replica
  - WRITE_TIMEOUT: "30s"
  - READ_TIMEOUT: "30s"
```

## Step 6: Monitor Function Performance

```bash
# View function metrics
curl http://openfaas-gateway:8080/system/functions | jq '.[] | select(.name == "process-image")'

# Check invocation count and replica count
```

## Conclusion

Serverless functions on Rancher provide a developer-friendly abstraction over Kubernetes pods while retaining full container customization. The choice of framework depends on your latency requirements and language preferences. OpenFaaS is the most practical starting point for teams new to serverless on Kubernetes.
