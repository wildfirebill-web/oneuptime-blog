# How to Use Dapr Agents with OpenAI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, OpenAI, GPT, LLM

Description: Learn how to configure and use Dapr Agents with OpenAI's GPT models, including tool calling, streaming responses, and managing API keys securely.

---

## Why Use Dapr Agents with OpenAI?

Dapr Agents provides a durable, stateful runtime for OpenAI-powered agents. While the OpenAI API handles LLM inference, Dapr handles the operational concerns - state persistence, retries, pub/sub messaging, and deployment on Kubernetes. This combination gives you production-grade AI agents without managing infrastructure complexity.

## Installation

```bash
pip install dapr-agents openai
```

## Configuring the OpenAI LLM Client

Dapr Agents wraps the OpenAI client with additional resiliency features:

```python
from dapr_agents.llm import OpenAIChat

llm = OpenAIChat(
    model="gpt-4o",
    api_key="sk-your-key",  # or use env var OPENAI_API_KEY
    temperature=0.7,
    max_tokens=2048,
    timeout=30
)
```

## Building an Agent with OpenAI

```python
import os
from dapr_agents import Agent, tool
from dapr_agents.llm import OpenAIChat

class CodeReviewAgent(Agent):
    name = "code-review-agent"
    instructions = """You are an expert code reviewer. Analyze code for bugs,
    security issues, performance problems, and style violations.
    Provide actionable feedback."""

    @tool
    def check_syntax(self, code: str, language: str) -> str:
        """Checks code syntax for the specified programming language.

        Args:
            code: The source code to check.
            language: Programming language (python, javascript, go, etc.)
        """
        # Integrate with language-specific linters
        return f"Syntax check for {language}: No critical errors found."

    @tool
    def search_vulnerabilities(self, code: str) -> str:
        """Scans code for known security vulnerabilities."""
        # Integrate with security scanning tools
        return "No known vulnerabilities detected."


agent = CodeReviewAgent(
    llm=OpenAIChat(
        model="gpt-4o",
        api_key=os.environ["OPENAI_API_KEY"]
    )
)

result = agent.run("""
Review this Python function:

def get_user(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    return db.execute(query)
""")

print(result)
```

## Using GPT-4o with Vision

For agents that process images:

```python
from dapr_agents.llm import OpenAIChat

llm = OpenAIChat(model="gpt-4o")

agent = ImageAnalysisAgent(llm=llm)
result = agent.run("Analyze this screenshot", images=["screenshot.png"])
```

## Streaming Responses

For long-running responses, enable streaming:

```python
from dapr_agents.llm import OpenAIChat

llm = OpenAIChat(
    model="gpt-4o",
    stream=True
)

agent = MyAgent(llm=llm)

for chunk in agent.stream("Explain quantum computing in detail"):
    print(chunk, end="", flush=True)
```

## Storing API Keys Securely with Dapr

Instead of hardcoding API keys, store them in a Dapr secret store:

```yaml
# components/secretstore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: secretstore
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
    - name: vaultName
      value: "my-key-vault"
```

Retrieve in your agent:

```python
from dapr import Client

dapr_client = Client()
secret = dapr_client.get_secret(
    store_name="secretstore",
    key="openai-api-key"
)

llm = OpenAIChat(
    model="gpt-4o",
    api_key=secret.secret["openai-api-key"]
)
```

## Handling Rate Limits

Dapr's resiliency policies handle OpenAI rate limits automatically:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: openai-resiliency
spec:
  policies:
    retries:
      openai-retry:
        policy: exponential
        maxRetries: 5
        initialInterval: 2s
        maxInterval: 60s
```

## Summary

Dapr Agents integrates with OpenAI through the `OpenAIChat` LLM client, supporting GPT-4o, vision, and streaming. Store API keys in Dapr secret stores for security, and use Dapr resiliency policies to handle rate limits with exponential backoff. The combination provides production-grade durability for OpenAI-powered agents.
