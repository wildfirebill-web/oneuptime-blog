# How to Use Dapr Agents for Multi-Agent Coordination

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, Multi-Agent, Coordination, Pub/Sub

Description: Learn how to coordinate multiple Dapr Agents using pub/sub messaging and Dapr's actor model for building scalable multi-agent AI systems.

---

## What Is Multi-Agent Coordination?

Multi-agent systems break complex tasks into specialized agents that collaborate by exchanging messages. In Dapr Agents, coordination happens through:

- **Pub/Sub messaging** - agents publish and subscribe to topics
- **Service invocation** - agents call each other directly
- **Shared state** - agents read and write to a common state store

This architecture enables parallel processing, specialization, and fault isolation between agents.

## Architecture Overview

A typical multi-agent pipeline has a coordinator agent that routes tasks to specialist agents:

```text
User Request
     |
Coordinator Agent
    /         \
Research      Writer
Agent         Agent
     \         /
   Final Response
```

## Defining Specialist Agents

Create each specialist agent as its own Dapr-enabled service:

```python
# research_agent.py
from dapr_agents import Agent, tool
from dapr_agents.messaging import DaprPubSubMessenger

class ResearchAgent(Agent):
    name = "research-agent"
    instructions = "You research topics and return structured findings."
    messenger = DaprPubSubMessenger(
        pubsub_name="pubsub",
        topic="research-results"
    )

    @tool
    def search_web(self, query: str) -> str:
        """Search the web for information on a topic."""
        # Call search API here
        return f"Search results for: {query}"

    async def handle_task(self, task: dict):
        result = self.run(task["query"])
        await self.messenger.publish({
            "task_id": task["id"],
            "result": result
        })
```

```python
# writer_agent.py
from dapr_agents import Agent, tool

class WriterAgent(Agent):
    name = "writer-agent"
    instructions = "You write clear, engaging content based on research findings."

    @tool
    def format_content(self, content: str, format: str = "blog") -> str:
        """Formats content into the specified format."""
        return f"Formatted as {format}: {content}"
```

## Coordinator Agent with Task Routing

```python
# coordinator_agent.py
from dapr_agents import Agent, tool
from dapr import Client

class CoordinatorAgent(Agent):
    name = "coordinator-agent"
    instructions = """You coordinate research and writing tasks.
    Break user requests into research and writing subtasks."""

    def __init__(self):
        super().__init__()
        self.dapr_client = Client()

    @tool
    def delegate_research(self, query: str, task_id: str) -> str:
        """Delegates a research task to the research agent."""
        self.dapr_client.publish_event(
            pubsub_name="pubsub",
            topic_name="research-tasks",
            data={"id": task_id, "query": query}
        )
        return f"Research task {task_id} delegated"

    @tool
    def delegate_writing(self, content: str, task_id: str) -> str:
        """Delegates a writing task to the writer agent."""
        self.dapr_client.publish_event(
            pubsub_name="pubsub",
            topic_name="writing-tasks",
            data={"id": task_id, "content": content}
        )
        return f"Writing task {task_id} delegated"
```

## Pub/Sub Component Configuration

Define the pub/sub component all agents share:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: "localhost:6379"
```

## Running Multiple Agents

Start each agent as a separate Dapr-enabled process:

```bash
# Terminal 1
dapr run --app-id coordinator-agent --app-port 8080 \
  --components-path ./components -- python coordinator_agent.py

# Terminal 2
dapr run --app-id research-agent --app-port 8081 \
  --components-path ./components -- python research_agent.py

# Terminal 3
dapr run --app-id writer-agent --app-port 8082 \
  --components-path ./components -- python writer_agent.py
```

## Subscribing to Task Topics

Each agent subscribes to its task topic:

```python
from dapr.ext.grpc import App

app = App()

@app.subscribe(pubsub_name="pubsub", topic="research-tasks")
def handle_research_task(event) -> None:
    data = json.loads(event.Data())
    agent = ResearchAgent()
    agent.handle_task(data)
```

## Monitoring Agent Coordination

Track agent interactions using Dapr distributed tracing:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: tracing-config
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://zipkin:9411/api/v2/spans"
```

## Summary

Dapr Agents supports multi-agent coordination through pub/sub messaging, service invocation, and shared state stores. Define specialist agents as separate Dapr services, use a coordinator to route tasks via pub/sub topics, and monitor coordination with Dapr's built-in tracing. This pattern enables scalable, fault-tolerant AI pipelines where agents can be scaled and replaced independently.
