# How to Use Dapr with AWS Bedrock for AI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, Bedrock, AI, Binding, Language Model

Description: Use Dapr output bindings to invoke AWS Bedrock foundation models for text generation and summarization from microservices without embedding Bedrock SDK calls.

---

AWS Bedrock provides access to foundation models from Anthropic, Meta, Amazon, and others via a unified API. Dapr's Bedrock binding lets microservices invoke AI models through the standard binding interface, decoupling AI logic from model-specific SDKs.

## Enable Bedrock Model Access

```bash
# List available foundation models
aws bedrock list-foundation-models \
  --region us-east-1 \
  --by-output-modality TEXT

# Request access to models in the AWS Console:
# Bedrock > Model access > Request model access
# Enable: Claude, Titan, Llama as needed
```

## Configure the Dapr Bedrock Binding

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: bedrock-binding
  namespace: default
spec:
  type: bindings.aws.bedrock
  version: v1
  metadata:
  - name: region
    value: us-east-1
  - name: model
    value: anthropic.claude-3-sonnet-20240229-v1:0
  - name: accessKey
    secretKeyRef:
      name: aws-credentials
      key: accessKey
  - name: secretKey
    secretKeyRef:
      name: aws-credentials
      key: secretKey
```

## Invoke a Foundation Model

```python
import requests
import json

def invoke_model(prompt: str, max_tokens: int = 512) -> str:
    resp = requests.post(
        "http://localhost:3500/v1.0/bindings/bedrock-binding",
        json={
            "operation": "invoke-model",
            "data": json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": max_tokens,
                "messages": [
                    {
                        "role": "user",
                        "content": prompt
                    }
                ]
            })
        }
    )
    resp.raise_for_status()
    result = resp.json()
    return result.get("content", [{}])[0].get("text", "")

# Generate a product description
description = invoke_model(
    "Write a 2-sentence product description for a wireless ergonomic keyboard."
)
print(description)
```

## Summarize Customer Feedback

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/summarize-feedback', methods=['POST'])
def summarize_feedback():
    body = request.json
    reviews = body.get('reviews', [])
    combined = "\n".join([f"- {r}" for r in reviews])

    prompt = f"""Summarize the following customer reviews in 3 bullet points:

{combined}

Provide only the bullet points, no introduction."""

    summary = invoke_model(prompt, max_tokens=256)
    return jsonify({"summary": summary})
```

## Use Amazon Titan for Embeddings

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: bedrock-titan
spec:
  type: bindings.aws.bedrock
  version: v1
  metadata:
  - name: region
    value: us-east-1
  - name: model
    value: amazon.titan-embed-text-v1
  - name: accessKey
    secretKeyRef:
      name: aws-credentials
      key: accessKey
  - name: secretKey
    secretKeyRef:
      name: aws-credentials
      key: secretKey
```

```python
def get_embedding(text: str) -> list:
    resp = requests.post(
        "http://localhost:3500/v1.0/bindings/bedrock-titan",
        json={
            "operation": "invoke-model",
            "data": json.dumps({"inputText": text})
        }
    )
    resp.raise_for_status()
    return resp.json().get("embedding", [])

embedding = get_embedding("ergonomic keyboard for developers")
print(f"Embedding dimensions: {len(embedding)}")
```

## IAM Policy for Bedrock

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": [
        "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0",
        "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v1"
      ]
    }
  ]
}
```

## Summary

Dapr's Bedrock binding provides a lightweight integration with AWS foundation models, allowing services to invoke Claude, Titan, and other models through the standard output binding API. This decouples AI capabilities from model-specific client code, making it straightforward to switch models or add AI features to existing services with minimal changes.
