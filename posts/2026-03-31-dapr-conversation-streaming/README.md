# How to Use Dapr Conversation API with Streaming Responses

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Conversation, Streaming, LLM, AI, Microservice

Description: Learn how to use streaming responses with the Dapr Conversation API, enabling real-time token-by-token output from LLMs for responsive user interfaces.

---

Streaming LLM responses dramatically improves perceived performance - users see text appearing in real time rather than waiting for the complete response. Dapr Conversation supports streaming mode for compatible providers, enabling token-by-token output delivery.

## Understanding Streaming Mode

Without streaming: client waits for the full LLM response (can take 5-30 seconds for long outputs).
With streaming: tokens appear as they are generated (first token in under 1 second).

## Enabling Streaming in the Component

Set the `stream` metadata in the component:

```yaml
# components/openai-streaming.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: openai-streaming
spec:
  type: conversation.openai
  version: v1
  metadata:
    - name: key
      secretKeyRef:
        name: openai-secret
        key: api-key
    - name: model
      value: "gpt-4o"
    - name: stream
      value: "true"
```

Or enable streaming per-request in the parameters:

```bash
curl -X POST http://localhost:3500/v1.0-alpha1/conversation/openai-streaming/converse \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": [{"message": "Write a 500-word essay on microservices", "role": "user"}],
    "parameters": {
      "stream": true
    }
  }'
```

## Streaming with Server-Sent Events (Node.js)

Build a streaming endpoint that proxies Dapr streaming responses to the browser:

```javascript
const express = require('express');
const app = express();
app.use(express.json());

const DAPR_URL = 'http://localhost:3500';

app.post('/api/stream-chat', async (req, res) => {
  const { message } = req.body;

  // Set up SSE headers
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();

  try {
    const daprResponse = await fetch(
      `${DAPR_URL}/v1.0-alpha1/conversation/openai-streaming/converse`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          inputs: [{ message, role: 'user' }],
          parameters: { stream: true }
        })
      }
    );

    // Stream the response back to the client
    const reader = daprResponse.body.getReader();
    const decoder = new TextDecoder();

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value);
      res.write(`data: ${chunk}\n\n`);
    }

    res.write('data: [DONE]\n\n');
    res.end();
  } catch (err) {
    res.write(`data: ${JSON.stringify({ error: err.message })}\n\n`);
    res.end();
  }
});

app.listen(6001);
```

## Browser Client for Streaming

```html
<!DOCTYPE html>
<html>
<head><title>Streaming Chat</title></head>
<body>
  <textarea id="message" placeholder="Ask something..."></textarea>
  <button onclick="sendMessage()">Send</button>
  <div id="response"></div>

  <script>
    async function sendMessage() {
      const message = document.getElementById('message').value;
      const responseDiv = document.getElementById('response');
      responseDiv.innerHTML = '';

      const response = await fetch('/api/stream-chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message })
      });

      const reader = response.body.getReader();
      const decoder = new TextDecoder();

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const text = decoder.decode(value);
        const lines = text.split('\n\n');

        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.slice(6);
            if (data === '[DONE]') break;
            try {
              const parsed = JSON.parse(data);
              if (parsed.outputs) {
                responseDiv.innerHTML += parsed.outputs[0].result;
              }
            } catch (e) {
              responseDiv.innerHTML += data;
            }
          }
        }
      }
    }
  </script>
</body>
</html>
```

## Streaming with Python

```python
import requests

def stream_conversation(component: str, message: str):
    with requests.post(
        f"http://localhost:3500/v1.0-alpha1/conversation/{component}/converse",
        json={
            "inputs": [{"message": message, "role": "user"}],
            "parameters": {"stream": True}
        },
        stream=True
    ) as response:
        for line in response.iter_lines():
            if line:
                decoded = line.decode('utf-8')
                if decoded.startswith('data:'):
                    chunk = decoded[5:].strip()
                    if chunk != '[DONE]':
                        print(chunk, end='', flush=True)

# Usage
stream_conversation("openai-streaming", "Explain async/await in JavaScript")
print()  # newline after completion
```

## Provider Streaming Support

Not all providers support streaming equally:

```yaml
OpenAI:      Full streaming support
Anthropic:   Full streaming support
Ollama:      Full streaming support (local)
AWS Bedrock: Streaming via InvokeModelWithResponseStream
Google AI:   Streaming supported
DeepSeek:    Streaming supported
```

## Summary

Dapr Conversation streaming support enables responsive user interfaces for LLM-powered applications. Enable streaming via the component `stream` metadata or per-request parameter, then proxy the stream to clients via Server-Sent Events for real-time token delivery in web applications.
