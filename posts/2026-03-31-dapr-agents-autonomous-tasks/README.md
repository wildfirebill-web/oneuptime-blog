# How to Use Dapr Agents for Autonomous Task Execution

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, Autonomous, Task Execution, AI

Description: Learn how to build autonomous AI agents with Dapr that execute multi-step tasks independently, make decisions, and complete goals without human intervention.

---

## What Is Autonomous Agent Execution?

An autonomous agent receives a high-level goal and independently plans and executes the steps needed to achieve it. Unlike simple Q&A agents, autonomous agents:

- Break goals into subtasks
- Execute tools in sequence or parallel
- React to results and adjust their plan
- Decide when the goal is complete
- Handle errors and retry failed steps

Dapr provides the durable execution infrastructure for long-running autonomous tasks.

## Building an Autonomous Research Agent

```python
from dapr_agents import Agent, tool
from dapr_agents.llm import OpenAIChat
from dapr import Client
import json
import time

class AutonomousResearchAgent(Agent):
    name = "autonomous-research-agent"
    instructions = """You are an autonomous research agent. When given a research goal:
    1. Break it into 3-5 specific research questions
    2. Research each question using available tools
    3. Synthesize findings into a comprehensive report
    4. Save the final report when complete

    Work independently. Do not ask for clarification - make reasonable assumptions.
    Mark the task complete when you have a comprehensive report."""

    max_iterations = 20  # Allow many tool calls for complex tasks

    @tool
    def web_search(self, query: str) -> str:
        """Searches the web for information on a specific query.

        Args:
            query: The search query to execute.
        """
        # Integrate with Serper, Tavily, or SerpAPI
        import httpx
        try:
            response = httpx.get(
                "https://api.tavily.com/search",
                json={"query": query, "max_results": 5},
                headers={"Authorization": "Bearer your-tavily-key"},
                timeout=15
            )
            results = response.json().get("results", [])
            summaries = [f"- {r['title']}: {r['content'][:200]}" for r in results[:3]]
            return "\n".join(summaries)
        except Exception as e:
            return f"Search failed: {str(e)}. Using general knowledge."

    @tool
    def read_webpage(self, url: str) -> str:
        """Reads and extracts text content from a webpage.

        Args:
            url: The URL of the webpage to read.
        """
        import httpx
        from html.parser import HTMLParser

        class TextExtractor(HTMLParser):
            def __init__(self):
                super().__init__()
                self.text = []
            def handle_data(self, data):
                self.text.append(data.strip())

        try:
            response = httpx.get(url, timeout=10, follow_redirects=True)
            parser = TextExtractor()
            parser.feed(response.text)
            text = " ".join(filter(None, parser.text))
            return text[:3000]
        except Exception as e:
            return f"Could not read {url}: {str(e)}"

    @tool
    def save_finding(self, category: str, finding: str) -> str:
        """Saves a research finding to the task's knowledge store.

        Args:
            category: The category or research question this addresses.
            finding: The finding or insight to save.
        """
        task_id = getattr(self, "_current_task_id", "default")
        key = f"finding-{task_id}-{category.replace(' ', '-').lower()}"
        Client().save_state("statestore", key, json.dumps({
            "category": category,
            "finding": finding,
            "timestamp": time.time()
        }))
        return f"Finding saved under category: {category}"

    @tool
    def compile_and_save_report(self, task_id: str, report: str) -> str:
        """Compiles findings into a final report and marks the task complete.

        Args:
            task_id: The unique task identifier.
            report: The complete research report text.
        """
        Client().save_state("statestore", f"report-{task_id}", json.dumps({
            "task_id": task_id,
            "report": report,
            "completed_at": time.time(),
            "status": "complete"
        }))
        # Publish completion event
        Client().publish_event(
            pubsub_name="pubsub",
            topic_name="task-completed",
            data=json.dumps({"task_id": task_id, "type": "research"})
        )
        return f"Report saved and task {task_id} marked complete."
```

## Submitting Autonomous Tasks

```python
from fastapi import FastAPI, BackgroundTasks
import uuid

app = FastAPI()

@app.post("/tasks/research")
async def create_research_task(request: dict, background: BackgroundTasks):
    task_id = str(uuid.uuid4())
    goal = request["goal"]

    # Save task metadata
    Client().save_state("statestore", f"task-{task_id}", json.dumps({
        "task_id": task_id,
        "goal": goal,
        "status": "running",
        "created_at": time.time()
    }))

    # Run autonomously in background
    background.add_task(run_autonomous_task, task_id, goal)
    return {"task_id": task_id, "status": "started"}

async def run_autonomous_task(task_id: str, goal: str):
    llm = OpenAIChat(model="gpt-4o")
    agent = AutonomousResearchAgent(llm=llm)
    agent._current_task_id = task_id

    try:
        agent.run(
            f"Task ID: {task_id}\n"
            f"Research Goal: {goal}\n\n"
            f"Work autonomously to complete this research goal. "
            f"Save your findings as you go, then compile and save the final report."
        )
    except Exception as e:
        Client().save_state("statestore", f"task-{task_id}", json.dumps({
            "task_id": task_id, "status": "failed", "error": str(e)
        }))
```

## Monitoring Autonomous Task Progress

```python
@app.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    state = Client().get_state("statestore", f"task-{task_id}")
    if not state.data:
        return {"error": "Task not found"}
    return json.loads(state.data)

@app.get("/tasks/{task_id}/report")
async def get_task_report(task_id: str):
    state = Client().get_state("statestore", f"report-{task_id}")
    if not state.data:
        return {"status": "not ready"}
    return json.loads(state.data)
```

## Safety Controls for Autonomous Agents

Limit what autonomous agents can do:

```python
class SafeAutonomousAgent(AutonomousResearchAgent):
    name = "safe-autonomous-agent"
    max_iterations = 15  # Prevent infinite loops

    ALLOWED_DOMAINS = ["wikipedia.org", "arxiv.org", "github.com"]

    @tool
    def web_search(self, query: str) -> str:
        """Searches the web (restricted to approved domains)."""
        # Override to add domain restrictions
        if len(query) > 200:
            return "Query too long. Please shorten it."
        return super().web_search(query)
```

## Summary

Autonomous Dapr Agents receive high-level goals and independently execute multi-step plans using tools like web search, page reading, and state storage. Run them asynchronously via background tasks to avoid blocking HTTP requests. Monitor progress through the Dapr state store, where agents save intermediate findings and final reports. Add safety controls by limiting `max_iterations` and restricting tool capabilities.
