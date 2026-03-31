# How to Use Dapr with AWS Lambda (via Container Images)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS Lambda, Serverless, Container Image, Cloud, AWS

Description: Learn how to run Dapr alongside an AWS Lambda function packaged as a container image, enabling Lambda functions to use Dapr's state store, pub/sub, and service invocation APIs.

---

AWS Lambda supports container images up to 10 GB, which opens the door to running multiple processes inside a Lambda container. By packaging the Dapr sidecar binary alongside your Lambda function handler, you can use Dapr's full API surface - state management, pub/sub, bindings, and secrets - from serverless Lambda functions. This approach trades the operational simplicity of Lambda's managed runtime for the portability and abstraction benefits of Dapr's building blocks.

## Architecture Overview

The pattern works by running the Dapr sidecar as a background process within the same Lambda container:

```text
Lambda Container
+-----------------------------------------+
|  Dapr Sidecar (background process)      |
|  - Listens on port 3500 (HTTP)          |
|  - Connects to external state store     |
+-----------------------------------------+
|  Lambda Handler (foreground process)    |
|  - Calls localhost:3500 for Dapr APIs   |
|  - Receives Lambda invocation events    |
+-----------------------------------------+
```

Limitations to be aware of:
- The Dapr sidecar must start within the Lambda init phase (up to 10 seconds)
- Lambda functions are stateless compute; actor placement and reminders are not suitable here
- State stores, pub/sub, bindings, and secrets work well
- Cold starts are longer because the Dapr process must initialize

## Creating the Dockerfile

```dockerfile
# Dockerfile
FROM public.ecr.aws/lambda/python:3.12

# Install Dapr sidecar binary
ARG DAPR_VERSION=1.14.0
RUN curl -fsSL https://raw.githubusercontent.com/dapr/cli/master/install/install.sh | bash && \
    dapr init --slim --runtime-version ${DAPR_VERSION} && \
    cp ~/.dapr/bin/daprd /usr/local/bin/daprd && \
    chmod +x /usr/local/bin/daprd

# Copy Dapr component definitions
COPY components/ /opt/dapr/components/

# Copy the Lambda function code
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py ${LAMBDA_TASK_ROOT}/
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Override the default entrypoint
ENTRYPOINT ["/entrypoint.sh"]
```

## Writing the Entrypoint Script

The entrypoint starts Dapr in the background before handing off to the Lambda runtime:

```bash
#!/bin/bash
# entrypoint.sh

set -e

echo "Starting Dapr sidecar..."
daprd \
  --app-id lambda-function \
  --app-port 9001 \
  --dapr-http-port 3500 \
  --components-path /opt/dapr/components \
  --log-level warn &

DAPR_PID=$!

# Wait for Dapr sidecar to be ready
echo "Waiting for Dapr sidecar to initialize..."
for i in $(seq 1 20); do
  if curl -sf http://localhost:3500/v1.0/healthz > /dev/null 2>&1; then
    echo "Dapr sidecar is ready"
    break
  fi
  echo "Waiting... attempt $i/20"
  sleep 0.5
done

# Start the Lambda runtime handler
echo "Starting Lambda runtime..."
exec python -m awslambdaric app.handler
```

## The Lambda Handler Using Dapr APIs

```python
# app.py
import json
import os
import requests

DAPR_PORT = int(os.environ.get("DAPR_HTTP_PORT", 3500))
DAPR_URL = f"http://localhost:{DAPR_PORT}/v1.0"

def save_state(key: str, value: dict) -> bool:
    """Save state via Dapr state store API."""
    response = requests.post(
        f"{DAPR_URL}/state/statestore",
        json=[{"key": key, "value": value}],
        timeout=5
    )
    return response.status_code == 204

def get_state(key: str) -> dict | None:
    """Retrieve state via Dapr state store API."""
    response = requests.get(
        f"{DAPR_URL}/state/statestore/{key}",
        timeout=5
    )
    if response.status_code == 200 and response.text:
        return response.json()
    return None

def publish_event(topic: str, data: dict) -> bool:
    """Publish an event via Dapr pub/sub."""
    response = requests.post(
        f"{DAPR_URL}/publish/pubsub/{topic}",
        json=data,
        timeout=5
    )
    return response.status_code == 204

def get_secret(store: str, key: str) -> dict | None:
    """Retrieve a secret via Dapr secrets API."""
    response = requests.get(
        f"{DAPR_URL}/secrets/{store}/{key}",
        timeout=5
    )
    if response.status_code == 200:
        return response.json()
    return None

def handler(event, context):
    """Lambda handler - process order events."""
    print(f"Processing event: {json.dumps(event)}")
    
    order_id = event.get("orderId", "unknown")
    customer_id = event.get("customerId", "unknown")
    
    # Save order state
    order_data = {
        "orderId": order_id,
        "customerId": customer_id,
        "status": "processing",
        "timestamp": str(context.aws_request_id)
    }
    
    if not save_state(f"order-{order_id}", order_data):
        return {"statusCode": 500, "body": "Failed to save state"}
    
    # Publish order-created event
    publish_event("orders", {"type": "order.created", "data": order_data})
    
    # Read it back to confirm
    saved = get_state(f"order-{order_id}")
    
    return {
        "statusCode": 200,
        "body": json.dumps({
            "orderId": order_id,
            "status": "created",
            "stateConfirmed": saved is not None
        })
    }
```

## Dapr Component for DynamoDB State Store

Use AWS DynamoDB as the Dapr state store so state persists across Lambda invocations:

```yaml
# components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.aws.dynamodb
  version: v1
  metadata:
  - name: table
    value: "dapr-lambda-state"
  - name: region
    value: "us-east-1"
  - name: ttlAttributeName
    value: "expiry"
```

Create the DynamoDB table:

```bash
aws dynamodb create-table \
  --table-name dapr-lambda-state \
  --attribute-definitions \
    AttributeName=key,AttributeType=S \
  --key-schema \
    AttributeName=key,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

## Building and Deploying to AWS Lambda

```bash
# Build the container image
docker build -t dapr-lambda:latest .

# Tag and push to ECR
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_REPO="${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/dapr-lambda"

aws ecr create-repository --repository-name dapr-lambda --region us-east-1
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ${ECR_REPO}

docker tag dapr-lambda:latest ${ECR_REPO}:latest
docker push ${ECR_REPO}:latest

# Create Lambda function from container image
aws lambda create-function \
  --function-name dapr-order-processor \
  --package-type Image \
  --code ImageUri=${ECR_REPO}:latest \
  --role arn:aws:iam::${AWS_ACCOUNT_ID}:role/lambda-execution-role \
  --timeout 30 \
  --memory-size 512 \
  --environment Variables="{DAPR_HTTP_PORT=3500}"

# Test the function
aws lambda invoke \
  --function-name dapr-order-processor \
  --payload '{"orderId":"test-1","customerId":"cust-42"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

cat response.json
```

## Summary

Running Dapr inside AWS Lambda container images enables serverless functions to leverage Dapr's portable building blocks for state, pub/sub, and secrets without being locked into AWS-specific SDKs. The key implementation details are: use an entrypoint script to start the Dapr sidecar before the Lambda runtime, wait for the sidecar health check before proceeding, and configure Dapr components to point at AWS-native services like DynamoDB and SNS/SQS. This pattern is best suited for workloads where portability matters more than minimizing cold-start latency.
