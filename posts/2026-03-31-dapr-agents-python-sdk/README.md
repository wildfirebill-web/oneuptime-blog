# How to Use Dapr Agents Python SDK

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, Python, SDK, LLM

Description: A comprehensive guide to the Dapr Agents Python SDK covering installation, agent classes, tool decorators, memory, messaging, and service patterns.

---

## What Is the Dapr Agents Python SDK?

The Dapr Agents Python SDK (`dapr-agents`) is the official Python library for building AI agents on Dapr. It provides:

- `Agent` base class with built-in tool calling loop
- `@tool` decorator for defining callable functions
- LLM client wrappers (OpenAI, Anthropic, Mistral, Bedrock, Ollama)
- Memory backends backed by Dapr state stores
- Messaging helpers for Dapr pub/sub
- `AgentService` for running agents as HTTP services

## Installation

```bash
pip install dapr-agents

# With specific LLM provider support
pip install dapr-agents[openai]       # OpenAI
pip install dapr-agents[anthropic]    # Anthropic
pip install dapr-agents[mistral]      # Mistral
pip install dapr-agents[all]          # All providers
```

## Core Agent Class

Every agent extends the `Agent` base class:

```python
from dapr_agents import Agent
from dapr_agents.llm import OpenAIChat

class MyAgent(Agent):
    name = "my-agent"                    # Dapr app ID
    description = "What this agent does" # Optional
    instructions = "System prompt here"  # LLM system message
    max_iterations = 10                   # Tool call loop limit
```

## Defining Tools with @tool

The `@tool` decorator exposes Python functions as LLM-callable tools:

```python
from dapr_agents import Agent, tool

class UtilityAgent(Agent):
    name = "utility-agent"
    instructions = "You help with utility tasks."

    @tool
    def fetch_url(self, url: str) -> str:
        """Fetches the content of a URL.

        Args:
            url: The URL to fetch.
        """
        import httpx
        response = httpx.get(url, timeout=10)
        return response.text[:2000]

    @tool
    def write_file(self, path: str, content: str) -> str:
        """Writes content to a file.

        Args:
            path: File path to write to.
            content: Content to write.
        """
        with open(path, "w") as f:
            f.write(content)
        return f"Written {len(content)} bytes to {path}"

    @tool
    def run_shell_command(self, command: str) -> str:
        """Runs a shell command and returns output. Use with caution.

        Args:
            command: The shell command to execute.
        """
        import subprocess
        result = subprocess.run(
            command, shell=True, capture_output=True, text=True, timeout=30
        )
        return result.stdout if result.returncode == 0 else result.stderr
```

## LLM Client Options

```python
from dapr_agents.llm import (
    OpenAIChat,
    AnthropicChat,
    MistralChat,
    AWSBedrockChat,
    HuggingFaceChat,
)

# OpenAI
openai_llm = OpenAIChat(model="gpt-4o", temperature=0.7)

# Anthropic
claude_llm = AnthropicChat(model="claude-3-5-sonnet-20241022")

# Mistral
mistral_llm = MistralChat(model="mistral-large-latest")

# AWS Bedrock
bedrock_llm = AWSBedrockChat(
    model="anthropic.claude-3-5-sonnet-20241022-v2:0",
    region_name="us-east-1"
)

# Ollama (local)
ollama_llm = OpenAIChat(
    model="llama3.2",
    base_url="http://localhost:11434/v1",
    api_key="ollama"
)
```

## Memory Backends

```python
from dapr_agents.memory import (
    InMemoryMemory,       # In-process only, lost on restart
    DaprStateMemory,      # Persistent, Dapr state store backed
)

class StatefulAgent(Agent):
    name = "stateful-agent"
    instructions = "You remember user preferences."
    memory = DaprStateMemory(
        store_name="statestore",
        session_id="user-123",   # Unique per user/session
        max_history=50
    )
```

## Running as an HTTP Service

```python
from dapr_agents import AgentService

service = AgentService(
    agent=MyAgent(llm=openai_llm),
    port=8080,
    health_path="/health"
)

if __name__ == "__main__":
    service.start()
```

Invoke via HTTP:

```bash
curl -X POST http://localhost:3500/v1.0/invoke/my-agent/method/run \
  -H "Content-Type: application/json" \
  -d '{"message": "Do something useful"}'
```

## Async Agent Execution

```python
import asyncio

async def main():
    agent = MyAgent(llm=openai_llm)
    result = await agent.run_async("Perform this task asynchronously")
    print(result)

asyncio.run(main())
```

## Summary

The Dapr Agents Python SDK provides the `Agent` base class, `@tool` decorator, multiple LLM client options, and `DaprStateMemory` for persistent conversation history. Run agents as HTTP services with `AgentService`, or invoke them programmatically with `agent.run()` or `agent.run_async()`. The SDK integrates with all Dapr components for state, pub/sub, and secrets.
