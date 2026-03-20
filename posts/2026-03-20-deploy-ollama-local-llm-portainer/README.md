# How to Deploy Ollama for Local LLM Inference via Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ollama, LLM, AI, Portainer, Docker, Local AI, Machine Learning, Privacy

Description: Deploy Ollama via Portainer to run large language models locally without sending data to external APIs, with GPU acceleration and an OpenAI-compatible REST interface.

---

Ollama lets you run LLMs like Llama 3, Mistral, Gemma, and Phi locally with a simple REST API that mirrors the OpenAI interface. Deploying it via Portainer gives your team a shared, managed LLM inference service that keeps data on-premises.

## Step 1: Deploy Ollama via Portainer Stack

For a CPU-only deployment:

```yaml
# ollama-stack.yml
version: "3.8"

services:
  ollama:
    image: ollama/ollama:0.1.27
    volumes:
      # Persist downloaded models — models can be several GB
      - ollama-data:/root/.ollama
    ports:
      - "11434:11434"    # Ollama REST API
    restart: unless-stopped
    networks:
      - ollama-net

volumes:
  ollama-data:

networks:
  ollama-net:
    driver: bridge
```

For GPU-accelerated inference, add NVIDIA device configuration:

```yaml
services:
  ollama:
    image: ollama/ollama:0.1.27
    volumes:
      - ollama-data:/root/.ollama
    ports:
      - "11434:11434"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    restart: unless-stopped
```

## Step 2: Pull a Model

After deploying, pull a model via the Portainer container console or via the API:

```bash
# Pull Llama 3 8B (4.7GB download)
curl http://localhost:11434/api/pull \
  -d '{"name": "llama3"}'

# Pull Mistral 7B
curl http://localhost:11434/api/pull \
  -d '{"name": "mistral"}'

# Pull a smaller model for resource-constrained environments
curl http://localhost:11434/api/pull \
  -d '{"name": "phi3:mini"}'

# List downloaded models
curl http://localhost:11434/api/tags
```

## Step 3: Run Inference

Ollama provides an OpenAI-compatible chat completions API:

```python
# ollama_chat.py — use Ollama's OpenAI-compatible API
from openai import OpenAI

# Point the OpenAI client at your local Ollama instance
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"   # API key is not validated, any value works
)

response = client.chat.completions.create(
    model="llama3",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain distributed tracing in two sentences."}
    ],
    stream=False
)

print(response.choices[0].message.content)
```

For streaming responses:

```python
# Streaming inference — useful for interactive UIs
response = client.chat.completions.create(
    model="llama3",
    messages=[{"role": "user", "content": "Write a haiku about containers."}],
    stream=True
)

for chunk in response:
    content = chunk.choices[0].delta.content
    if content:
        print(content, end="", flush=True)
```

## Step 4: Deploy Multiple Models

Manage multiple models from a single Ollama instance:

```bash
# The Ollama container auto-selects the GPU when multiple models are loaded
# Models are loaded on demand and unloaded after a period of inactivity

# Customize model parameters with a Modelfile
cat > /tmp/custom-llama3.modelfile << 'EOF'
FROM llama3

# Set the system prompt
SYSTEM """You are a DevOps expert specializing in containers and Kubernetes."""

# Adjust generation parameters
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER num_ctx 4096
EOF

# Create the custom model via the API
curl http://localhost:11434/api/create \
  -d "$(jq -Rs '{name: "devops-llama3", modelfile: .}' /tmp/custom-llama3.modelfile)"
```

## Step 5: Resource Planning

| Model | RAM Required | GPU VRAM | Inference Speed (CPU) |
|-------|-------------|----------|----------------------|
| phi3:mini (3.8B) | 4GB | 4GB | ~10 tokens/s |
| mistral (7B) | 8GB | 8GB | ~5 tokens/s |
| llama3 (8B) | 8GB | 8GB | ~5 tokens/s |
| llama3:70b | 40GB | 40GB | ~1 token/s |

## Summary

Ollama deployed via Portainer gives your team a private, on-premises LLM service that works with the same OpenAI API your code already uses. All data stays local, latency is predictable, and Portainer handles the container lifecycle.
