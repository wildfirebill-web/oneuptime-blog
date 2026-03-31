# How to Use Dapr with AWS Lambda via Container Images

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS Lambda, Container, Serverless, Cloud

Description: Run Dapr alongside AWS Lambda using container image packaging to access Dapr state stores, pub/sub, and secrets from serverless functions.

---

## Overview

AWS Lambda supports container images up to 10GB, making it possible to include the Dapr sidecar binary alongside your function code. While Lambda's ephemeral execution model doesn't map perfectly to Dapr's sidecar model, you can use the Dapr HTTP API directly in init code.

## Architecture Approach

Since Lambda doesn't support sidecars natively, the pattern is:
1. Bundle `daprd` binary in the container image
2. Start `daprd` as a background process in the Lambda init phase
3. Call Dapr HTTP API from your handler
4. Use a persistent backing store (Redis on ElastiCache, DynamoDB)

## Dockerfile for Lambda with Dapr

```dockerfile
FROM public.ecr.aws/lambda/nodejs:20

# Install Dapr CLI
RUN curl -fsSL https://raw.githubusercontent.com/dapr/cli/master/install/install.sh | \
    /bin/bash -s 1.13.0

# Copy Dapr runtime
COPY --from=daprio/daprd:1.13.0 /daprd /usr/local/bin/daprd

# Copy Dapr components
COPY components/ /components/

# Copy function code
COPY index.js ${LAMBDA_TASK_ROOT}/

CMD ["index.handler"]
```

## Starting Dapr in the Lambda Init Phase

```javascript
// index.js
const { execSync, spawn } = require('child_process');
const http = require('http');

let daprStarted = false;

async function startDapr() {
    if (daprStarted) return;

    // Start daprd in background
    const dapr = spawn('daprd', [
        '--app-id', 'lambda-function',
        '--app-port', '8080',
        '--dapr-http-port', '3500',
        '--components-path', '/components',
        '--log-level', 'warn'
    ], { detached: true });

    // Wait for Dapr to be ready
    await waitForDapr();
    daprStarted = true;
}

async function waitForDapr(maxWaitMs = 5000) {
    const start = Date.now();
    while (Date.now() - start < maxWaitMs) {
        try {
            await fetch('http://localhost:3500/v1.0/healthz');
            return;
        } catch {
            await new Promise(r => setTimeout(r, 100));
        }
    }
    throw new Error('Dapr failed to start within timeout');
}

exports.handler = async (event) => {
    await startDapr();

    // Use Dapr state store
    const key = event.orderId;
    const stateResp = await fetch(
        `http://localhost:3500/v1.0/state/statestore/${key}`
    );
    const state = await stateResp.json();

    return { statusCode: 200, body: JSON.stringify(state) };
};
```

## Dapr Component for ElastiCache Redis

```yaml
# components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "my-elasticache.xxxxx.0001.use1.cache.amazonaws.com:6379"
  - name: enableTLS
    value: "true"
```

## Lambda Execution Role for Secrets

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue"
    ],
    "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:dapr/*"
  }]
}
```

## Performance Considerations

Lambda cold starts include Dapr initialization time (~500ms-1s). Optimize with:

```javascript
// Use provisioned concurrency to keep Dapr warm
// In serverless.yml or SAM template:
```

```yaml
# serverless.yml
functions:
  orderHandler:
    image: my-lambda-dapr:latest
    provisionedConcurrency: 5
```

## Alternative: Dapr HTTP Client without Sidecar

For simpler scenarios, call Dapr-backed services via HTTP directly:

```javascript
// Call a Dapr-enabled ECS service from Lambda
const resp = await fetch(
    'http://order-service.internal/v1.0/state/statestore/order-123'
);
```

## Summary

Dapr can run alongside AWS Lambda functions using container image packaging by bundling the `daprd` binary and starting it during Lambda initialization. Use provisioned concurrency to amortize startup costs, connect to persistent backing stores on ElastiCache or DynamoDB, and cache the Dapr process across warm invocations. For simpler use cases, consider calling Dapr-enabled ECS or EKS services from Lambda via HTTP instead of running the sidecar in-process.
