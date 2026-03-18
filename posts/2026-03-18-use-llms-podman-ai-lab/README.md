# How to Use LLMs with Podman AI Lab

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, AI, LLM, Machine Learning, Natural Language Processing

Description: Learn how to work with large language models in Podman AI Lab for text generation, code assistance, and conversational AI.

---

> Large language models running locally through Podman AI Lab give you GPT-like capabilities without API keys or usage fees.

Large Language Models (LLMs) are the backbone of modern AI applications like chatbots, code assistants, and content generators. Podman AI Lab lets you run these models entirely on your local hardware. This guide covers selecting the right LLM for your task, configuring it for optimal performance, and integrating it into common workflows.

---

## Understanding Available LLMs

```bash
# Podman AI Lab catalog includes several LLM families.
# Each family has different strengths:

# Llama (Meta) - General purpose, strong reasoning
# - llama-3-8b-instruct: Latest generation, excellent quality
# - llama-2-7b-chat: Older but well-tested, lower resource needs

# Mistral - Efficient, strong performance for size
# - mistral-7b-instruct: Excellent quality-to-size ratio
# - mixtral-8x7b: Mixture of experts, higher quality but larger

# Granite (IBM) - Enterprise-focused
# - granite-7b-instruct: Trained on enterprise data
# - granite-code-3b: Lightweight code model

# CodeLlama (Meta) - Code-focused
# - codellama-7b-instruct: Code generation and explanation
```

## Starting an LLM Service

```bash
# Start a general-purpose LLM server
podman run -d \
  --name llm-server \
  -p 8080:8080 \
  -v ~/ai-models:/models:ro \
  ghcr.io/ggerganov/llama.cpp:server \
  --model /models/mistral-7b-instruct-q4_k_m.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  --ctx-size 4096 \
  --threads 4

# Wait for the model to load
podman logs -f llm-server 2>&1 | grep -m1 "listening"
```

## Text Generation Use Cases

### Conversational Chat

```bash
# Multi-turn conversation with system prompt
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a senior Linux administrator. Be concise and practical."},
      {"role": "user", "content": "How do I find which process is using the most memory?"},
      {"role": "assistant", "content": "Use: ps aux --sort=-%mem | head -10"},
      {"role": "user", "content": "How do I kill the top one safely?"}
    ],
    "temperature": 0.3,
    "max_tokens": 256
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
```

### Code Generation

```bash
# Generate a Python script from a description
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are an expert Python developer. Write clean, well-commented code."},
      {"role": "user", "content": "Write a Python script that monitors disk usage and sends an alert if any partition exceeds 90% usage."}
    ],
    "temperature": 0.2,
    "max_tokens": 1024
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
```

### Text Summarization

```bash
# Summarize a long text
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "Summarize the following text in 3 bullet points. Be concise."},
      {"role": "user", "content": "Containers are lightweight, portable units of software that package an application and its dependencies together. Unlike virtual machines, containers share the host operating system kernel, making them much more efficient in terms of resource usage. Container technologies like Podman and Docker have revolutionized software deployment by ensuring applications run consistently across different environments. The adoption of containers has led to the development of orchestration platforms like Kubernetes, which manage the lifecycle of containerized applications at scale."}
    ],
    "temperature": 0.3,
    "max_tokens": 256
  }' | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
```

## Building an LLM Pipeline Script

```bash
cat << 'SCRIPT' > ~/llm_pipeline.sh
#!/bin/bash
# A reusable script for sending prompts to the local LLM server

LLM_URL="${LLM_URL:-http://localhost:8080}"
SYSTEM_PROMPT="${SYSTEM_PROMPT:-You are a helpful assistant.}"
TEMPERATURE="${TEMPERATURE:-0.7}"
MAX_TOKENS="${MAX_TOKENS:-512}"

# Read prompt from argument or stdin
if [ -n "$1" ]; then
  PROMPT="$1"
else
  PROMPT=$(cat)
fi

# Send the request and extract the response text
curl -s "${LLM_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d "$(python3 -c "
import json
print(json.dumps({
    'messages': [
        {'role': 'system', 'content': '''${SYSTEM_PROMPT}'''},
        {'role': 'user', 'content': '''${PROMPT}'''}
    ],
    'temperature': ${TEMPERATURE},
    'max_tokens': ${MAX_TOKENS}
}))")" | python3 -c "import sys,json; print(json.load(sys.stdin)['choices'][0]['message']['content'])"
SCRIPT
chmod +x ~/llm_pipeline.sh

# Usage examples:
# Direct prompt
~/llm_pipeline.sh "Explain DNS in one paragraph"

# Pipe content into the LLM
echo "Review this Containerfile for security issues: FROM ubuntu:latest\nRUN apt-get update" | \
  SYSTEM_PROMPT="You are a container security expert." ~/llm_pipeline.sh

# Summarize a file
cat /etc/os-release | SYSTEM_PROMPT="Summarize this system information." ~/llm_pipeline.sh
```

## Optimizing LLM Performance

```bash
# Monitor tokens per second during generation
podman logs llm-server 2>&1 | grep -E "tokens per second|eval time"

# Key tuning parameters for CPU inference:
# --threads: Match your CPU core count (not hyperthreads)
# --ctx-size: Reduce if memory is tight (2048 minimum for chat)
# --batch-size: Increase for better throughput (256-1024)
# --mlock: Lock model in memory to prevent swapping

# Restart with optimized settings
podman stop llm-server && podman rm llm-server

podman run -d \
  --name llm-server \
  -p 8080:8080 \
  -v ~/ai-models:/models:ro \
  --cpus 8 --memory 10g \
  ghcr.io/ggerganov/llama.cpp:server \
  --model /models/mistral-7b-instruct-q4_k_m.gguf \
  --host 0.0.0.0 --port 8080 \
  --ctx-size 4096 --threads 8 --batch-size 512 --mlock
```

## Cleaning Up

```bash
# Stop and remove the LLM server
podman stop llm-server && podman rm llm-server
```

## Summary

Podman AI Lab makes it practical to run LLMs locally for a wide range of tasks including chat, code generation, and text summarization. Choosing the right model family and quantization level for your hardware is key to getting good results. By wrapping LLM calls in scripts, you can integrate local AI into your existing DevOps workflows. The OpenAI-compatible API means any tool or library that works with OpenAI will work with your local model server.
