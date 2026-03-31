# How to Use Dapr Agents with Mistral

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, Mistral, LLM, AI

Description: Learn how to configure Dapr Agents with Mistral AI models for efficient, cost-effective AI agent deployments with strong multilingual and code generation capabilities.

---

## Why Mistral with Dapr Agents?

Mistral AI offers a range of models from the efficient Mistral 7B to the powerful Mistral Large, with strong performance in multilingual tasks and code generation. When cost efficiency matters, Mistral models provide excellent performance-per-dollar. Paired with Dapr's operational infrastructure, you get reliable agent execution without overpaying for inference.

## Installation

```bash
pip install dapr-agents mistralai
```

## Configuring the Mistral LLM Client

```python
from dapr_agents.llm import MistralChat

llm = MistralChat(
    model="mistral-large-latest",
    api_key="your-mistral-api-key",  # or MISTRAL_API_KEY env var
    temperature=0.3,
    max_tokens=4096
)
```

Available models:
- `mistral-small-latest` - Fast and efficient
- `mistral-medium-latest` - Balanced performance
- `mistral-large-latest` - Most capable

## Building a Code Generation Agent

Mistral excels at code tasks. Here is a code generation agent:

```python
import os
from dapr_agents import Agent, tool
from dapr_agents.llm import MistralChat

class CodeGenAgent(Agent):
    name = "codegen-agent"
    instructions = """You are an expert software engineer specializing in
    Python and Go. Generate clean, well-documented, production-ready code.
    Always include error handling and tests."""

    @tool
    def create_file(self, filename: str, content: str) -> str:
        """Creates a new code file with the specified content.

        Args:
            filename: The name of the file to create.
            content: The code content to write.
        """
        with open(filename, "w") as f:
            f.write(content)
        return f"Created {filename} ({len(content)} bytes)"

    @tool
    def run_tests(self, test_file: str) -> str:
        """Runs Python unit tests in a test file.

        Args:
            test_file: Path to the test file to run.
        """
        import subprocess
        result = subprocess.run(
            ["python", "-m", "pytest", test_file, "-v"],
            capture_output=True, text=True, timeout=60
        )
        return result.stdout if result.returncode == 0 else result.stderr


llm = MistralChat(
    model="mistral-large-latest",
    api_key=os.environ["MISTRAL_API_KEY"]
)

agent = CodeGenAgent(llm=llm)
result = agent.run(
    "Create a Python function that calculates the Fibonacci sequence "
    "using memoization, with unit tests."
)
print(result)
```

## Multilingual Agent with Mistral

Mistral handles European languages particularly well:

```python
class MultilingualSupportAgent(Agent):
    name = "support-agent"
    instructions = """You are a multilingual customer support agent.
    Detect the user's language and respond in the same language.
    Support French, Spanish, German, Italian, and English."""

    @tool
    def detect_language(self, text: str) -> str:
        """Detects the language of the input text."""
        # Use a language detection library
        from langdetect import detect
        return detect(text)

    @tool
    def translate_response(self, text: str, target_lang: str) -> str:
        """Translates a response to the target language."""
        # Integrate translation API
        return f"[Translated to {target_lang}]: {text}"
```

## Using Mistral Le Chat / Self-Hosted

For self-hosted Mistral (via vLLM or Ollama):

```python
from dapr_agents.llm import MistralChat

llm = MistralChat(
    model="mistral-7b-instruct",
    base_url="http://your-vllm-server:8000/v1",
    api_key="not-needed"
)
```

## Function Calling with Mistral

Mistral's function calling is compatible with the OpenAI tool format:

```python
from dapr_agents import Agent, tool
from dapr_agents.llm import MistralChat

class CalculatorAgent(Agent):
    name = "calculator-agent"

    @tool
    def add(self, a: float, b: float) -> float:
        """Adds two numbers together."""
        return a + b

    @tool
    def multiply(self, a: float, b: float) -> float:
        """Multiplies two numbers together."""
        return a * b

llm = MistralChat(model="mistral-large-latest")
agent = CalculatorAgent(llm=llm)
result = agent.run("What is 15.5 multiplied by 3 and then added to 42?")
```

## Running with Dapr

```bash
export MISTRAL_API_KEY="your-key"

dapr run --app-id codegen-agent \
  --app-port 8080 \
  --components-path ./components \
  -- python agent.py
```

## Summary

Dapr Agents integrates with Mistral AI through the `MistralChat` LLM client. Mistral Large is ideal for complex code generation and reasoning, while Mistral Small offers cost-effective inference for simpler tasks. Mistral's multilingual capabilities make it a strong choice for global applications, and its OpenAI-compatible function calling works seamlessly with Dapr Agents' `@tool` decorator.
