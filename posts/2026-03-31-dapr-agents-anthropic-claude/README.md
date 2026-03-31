# How to Use Dapr Agents with Anthropic Claude

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, Anthropic, Claude, LLM

Description: Learn how to configure Dapr Agents to use Anthropic's Claude models, including tool use, extended thinking, and secure API key management.

---

## Why Use Claude with Dapr Agents?

Anthropic's Claude models offer strong reasoning, long context windows, and reliable tool use. Paired with Dapr Agents, you get durable state management, pub/sub coordination, and Kubernetes-native deployment for Claude-powered agents.

## Installation

```bash
pip install dapr-agents anthropic
```

## Configuring the Anthropic LLM Client

```python
from dapr_agents.llm import AnthropicChat

llm = AnthropicChat(
    model="claude-3-5-sonnet-20241022",
    api_key="sk-ant-your-key",  # or ANTHROPIC_API_KEY env var
    max_tokens=4096,
    temperature=0.5
)
```

## Building an Analysis Agent with Claude

```python
import os
from dapr_agents import Agent, tool
from dapr_agents.llm import AnthropicChat

class AnalysisAgent(Agent):
    name = "analysis-agent"
    instructions = """You are a data analysis expert. Use available tools
    to analyze data, identify trends, and provide clear insights.
    Always explain your reasoning step by step."""

    @tool
    def load_csv(self, filepath: str) -> str:
        """Loads and parses a CSV file for analysis.

        Args:
            filepath: Path to the CSV file to load.
        """
        import csv
        rows = []
        with open(filepath) as f:
            reader = csv.DictReader(f)
            rows = list(reader)
        return f"Loaded {len(rows)} rows with columns: {list(rows[0].keys()) if rows else []}"

    @tool
    def calculate_statistics(self, column: str, data: str) -> str:
        """Calculates basic statistics for a numeric column.

        Args:
            column: Name of the column to analyze.
            data: JSON string of data values.
        """
        import json
        import statistics
        values = [float(v) for v in json.loads(data) if v]
        return (
            f"Column '{column}': "
            f"mean={statistics.mean(values):.2f}, "
            f"median={statistics.median(values):.2f}, "
            f"stdev={statistics.stdev(values):.2f}"
        )


llm = AnthropicChat(
    model="claude-3-5-sonnet-20241022",
    api_key=os.environ["ANTHROPIC_API_KEY"]
)

agent = AnalysisAgent(llm=llm)
result = agent.run("Analyze the sales data in sales_q1.csv and identify the top performing products.")
print(result)
```

## Using Claude's Extended Thinking

Claude 3.7 Sonnet supports extended thinking mode for complex reasoning:

```python
from dapr_agents.llm import AnthropicChat

llm = AnthropicChat(
    model="claude-3-7-sonnet-20250219",
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    max_tokens=16000
)

agent = ComplexReasoningAgent(llm=llm)
result = agent.run("Solve this multi-step logistics optimization problem...")
```

## Handling Claude's Long Context Window

Claude supports up to 200,000 token context windows. For document analysis agents:

```python
@tool
def analyze_document(self, document_path: str) -> str:
    """Analyzes a long document using Claude's extended context window."""
    with open(document_path) as f:
        content = f.read()
    # Claude can handle very large documents directly
    return f"Document length: {len(content)} chars - ready for analysis"
```

Configure the LLM with a high token limit:

```python
llm = AnthropicChat(
    model="claude-3-5-sonnet-20241022",
    max_tokens=8192
)
```

## Storing Anthropic API Keys in Dapr

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: secretstore
spec:
  type: secretstores.local.file
  version: v1
  metadata:
    - name: secretsFile
      value: "./secrets.json"
```

`secrets.json`:

```json
{
  "anthropic-api-key": "sk-ant-your-key-here"
}
```

```python
from dapr import Client

client = Client()
secret = client.get_secret("secretstore", "anthropic-api-key")

llm = AnthropicChat(
    model="claude-3-5-sonnet-20241022",
    api_key=secret.secret["anthropic-api-key"]
)
```

## Running with Dapr

```bash
export ANTHROPIC_API_KEY="sk-ant-your-key"

dapr run --app-id analysis-agent \
  --app-port 8080 \
  --components-path ./components \
  -- python agent.py
```

## Summary

Dapr Agents supports Anthropic Claude through the `AnthropicChat` LLM client. Use Claude 3.5 Sonnet for tool-heavy agent tasks and Claude 3.7 Sonnet with extended thinking for complex multi-step reasoning. Leverage Claude's 200K token context window for document analysis agents, and secure API keys using Dapr secret stores.
