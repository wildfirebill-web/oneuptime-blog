# How to Build Your First AI Agent with Dapr Agents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Agent, AI, Python, LLM

Description: Step-by-step tutorial for building your first AI agent with Dapr Agents, including tool definition, state management, and running the agent end to end.

---

## Project Setup

Start by creating a project directory and installing dependencies:

```bash
mkdir my-first-agent && cd my-first-agent
python -m venv venv && source venv/bin/activate
pip install dapr-agents openai
```

Create a `components` directory for Dapr configuration:

```bash
mkdir components
```

## Define a State Store Component

Agents need a state store to persist conversation history. Create a Redis state store component:

```yaml
# components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: "localhost:6379"
    - name: redisPassword
      value: ""
```

## Write the Agent

Create the main agent file:

```python
# main.py
import os
from dapr_agents import Agent, tool
from dapr_agents.llm import OpenAIChat

class WeatherAgent(Agent):
    name = "weather-agent"
    instructions = """You are a weather assistant. Use the available tools
    to answer questions about weather. Always provide helpful context."""

    @tool
    def get_weather(self, city: str) -> str:
        """Retrieves current weather for a city.

        Args:
            city: The name of the city to get weather for.
        """
        # In a real agent, call a weather API here
        weather_data = {
            "London": "15C, cloudy with light rain",
            "New York": "22C, sunny",
            "Tokyo": "18C, partly cloudy",
        }
        return weather_data.get(city, f"Weather data for {city} is unavailable.")

    @tool
    def get_forecast(self, city: str, days: int = 3) -> str:
        """Returns a multi-day weather forecast.

        Args:
            city: The name of the city.
            days: Number of days to forecast (1-7).
        """
        return f"{days}-day forecast for {city}: Mild temperatures with occasional showers."


if __name__ == "__main__":
    llm = OpenAIChat(
        model="gpt-4o",
        api_key=os.environ["OPENAI_API_KEY"]
    )

    agent = WeatherAgent(llm=llm)

    print("Weather Agent ready. Type your question:")
    question = input("> ")
    response = agent.run(question)
    print(f"\nAgent: {response}")
```

## Run the Agent with Dapr

```bash
export OPENAI_API_KEY="sk-your-key-here"

dapr run --app-id weather-agent \
  --app-port 8080 \
  --dapr-http-port 3500 \
  --components-path ./components \
  -- python main.py
```

Expected interaction:

```yaml
Weather Agent ready. Type your question:
> What is the weather in London and should I bring an umbrella?

Agent: The current weather in London is 15C with cloudy skies and light rain.
Yes, you should definitely bring an umbrella! The conditions suggest ongoing
rain, and the 3-day forecast also predicts occasional showers.
```

## Adding Agent Memory

To make the agent remember previous messages:

```python
from dapr_agents import Agent, ConversationHistory
from dapr_agents.memory import DaprStateMemory

class WeatherAgent(Agent):
    name = "weather-agent"
    instructions = "You are a weather assistant."
    memory = DaprStateMemory(
        store_name="statestore",
        max_history=20
    )
```

## Handling Errors Gracefully

Add error handling for tool failures:

```python
@tool
def get_weather(self, city: str) -> str:
    """Retrieves weather for a city."""
    try:
        response = requests.get(
            f"https://api.weather.com/current?city={city}",
            timeout=5
        )
        response.raise_for_status()
        return response.json()["description"]
    except requests.RequestException as e:
        return f"Unable to retrieve weather for {city}: {str(e)}"
```

## Running as a Service

To run the agent as a persistent HTTP service:

```python
from dapr_agents import AgentService

service = AgentService(agent=WeatherAgent())
service.start(port=8080)
```

Then invoke it via HTTP:

```bash
curl -X POST http://localhost:3500/v1.0/invoke/weather-agent/method/run \
  -H "Content-Type: application/json" \
  -d '{"message": "What is the weather in Tokyo?"}'
```

## Summary

Building a Dapr Agent involves defining a class extending `Agent`, decorating tool methods with `@tool`, and running with `dapr run`. Dapr manages state persistence automatically using the configured state store. Add `DaprStateMemory` for multi-turn conversation history, and wrap tools in try/except blocks for production reliability.
