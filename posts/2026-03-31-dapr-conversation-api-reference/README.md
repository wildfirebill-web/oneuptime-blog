# How to Use the Dapr Conversation API Reference

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Conversation, API, LLM, AI

Description: A practical reference for the Dapr Conversation API covering how to interact with large language models through a unified, provider-agnostic interface.

---

## Overview

The Dapr Conversation API provides a uniform interface for interacting with large language models (LLMs) including OpenAI GPT, Anthropic Claude, and Azure OpenAI. Applications send conversation messages and receive responses through the Dapr sidecar, which handles provider-specific authentication and protocols.

## Supported LLM Providers

- OpenAI (GPT-4, GPT-3.5-turbo)
- Azure OpenAI
- Anthropic Claude
- AWS Bedrock
- Hugging Face

## Component Definition: OpenAI

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: openai
spec:
  type: conversation.openai
  version: v1
  metadata:
    - name: key
      secretKeyRef:
        name: openai-secret
        key: apiKey
    - name: model
      value: gpt-4
    - name: cachingEnabled
      value: "true"
```

## Component Definition: Anthropic

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: claude
spec:
  type: conversation.anthropic
  version: v1
  metadata:
    - name: key
      secretKeyRef:
        name: anthropic-secret
        key: apiKey
    - name: model
      value: claude-3-5-sonnet-20241022
```

## Sending a Conversation Request

**POST** `/v1.0-alpha1/conversation/{componentName}/converse`

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/conversation/openai/converse \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [
      {
        "message": "Summarize this customer complaint in one sentence: The order arrived damaged and customer service was unhelpful.",
        "role": "user"
      }
    ],
    "parameters": {
      "temperature": 0.3,
      "maxTokens": 100
    }
  }'
```

Response:

```json
{
  "outputs": [
    {
      "result": "A customer received a damaged order and found customer service unresponsive.",
      "parameters": {}
    }
  ]
}
```

## Multi-Turn Conversation

Include prior conversation turns to maintain context:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/conversation/openai/converse \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [
      {"message": "My name is Alice.", "role": "user"},
      {"message": "Hello Alice! How can I help you today?", "role": "assistant"},
      {"message": "What did I just tell you my name was?", "role": "user"}
    ]
  }'
```

## Using the SDK (Python)

```python
from dapr.clients import DaprClient
from dapr.clients.grpc._request import ConversationInput

with DaprClient() as client:
    inputs = [
        ConversationInput(
            content="Extract all product names from this text: We ordered a MacBook Pro and an iPhone 15.",
            role="user"
        )
    ]

    response = client.converse_alpha1(
        name="openai",
        inputs=inputs,
        temperature=0.0,
        context_id="session-user-123"
    )

    print(response.outputs[0].result)
```

## Enabling PII Scrubbing

The Dapr Conversation API can automatically remove personally identifiable information before sending to the LLM:

```yaml
metadata:
  - name: cachingEnabled
    value: "true"
  - name: piiScrubbingEnabled
    value: "true"
```

## Conversation Caching

Enable response caching to reduce API costs for repeated identical queries:

```yaml
metadata:
  - name: cachingEnabled
    value: "true"
  - name: cacheSize
    value: "1000"
```

## Summary

The Dapr Conversation API makes LLM integration portable across providers. Switching from OpenAI to Anthropic only requires changing the component configuration, not application code. Built-in caching and PII scrubbing features reduce costs and compliance risk, making it suitable for enterprise applications that need to interact with multiple AI providers.
