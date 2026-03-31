# How to Use Dapr MCP (Model Context Protocol) Integration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, MCP, Model Context Protocol, Agent, LLM

Description: Learn how to integrate Dapr with Anthropic's Model Context Protocol (MCP) to expose Dapr services as MCP tools and resources for AI applications.

---

## What Is MCP?

The Model Context Protocol (MCP) is an open standard from Anthropic that defines how AI applications connect to external tools and data sources. MCP servers expose tools, resources, and prompts that MCP clients (Claude Desktop, AI agents, IDE extensions) can discover and invoke.

Dapr's MCP integration lets you expose Dapr components and services as MCP tools, enabling any MCP-compatible AI application to interact with your Dapr infrastructure.

## Installing Dapr MCP Support

```bash
pip install dapr-agents[mcp]
# or
pip install dapr-mcp
```

## Creating a Dapr MCP Server

Expose Dapr state store and pub/sub operations as MCP tools:

```python
from dapr_agents.mcp import DaprMCPServer
from dapr import Client

server = DaprMCPServer(name="dapr-tools-server")
dapr_client = Client()

@server.tool("save_state")
def save_state(key: str, value: str, store_name: str = "statestore") -> str:
    """Save a key-value pair to the Dapr state store.

    Args:
        key: The key to store the value under.
        value: The value to store.
        store_name: Name of the Dapr state store component.
    """
    dapr_client.save_state(store_name, key, value)
    return f"Saved '{key}' to {store_name}"

@server.tool("get_state")
def get_state(key: str, store_name: str = "statestore") -> str:
    """Retrieve a value from the Dapr state store.

    Args:
        key: The key to retrieve.
        store_name: Name of the Dapr state store component.
    """
    result = dapr_client.get_state(store_name, key)
    return result.data.decode() if result.data else "Key not found"

@server.tool("publish_event")
def publish_event(topic: str, data: str, pubsub_name: str = "pubsub") -> str:
    """Publish an event to a Dapr pub/sub topic.

    Args:
        topic: The topic name to publish to.
        data: The event data as a JSON string.
        pubsub_name: Name of the Dapr pub/sub component.
    """
    dapr_client.publish_event(pubsub_name, topic, data)
    return f"Event published to topic '{topic}'"

@server.tool("invoke_service")
def invoke_service(app_id: str, method: str, data: str = "") -> str:
    """Invoke a method on a Dapr-enabled service.

    Args:
        app_id: The Dapr application ID of the target service.
        method: The method name to invoke.
        data: Optional request body as a string.
    """
    response = dapr_client.invoke_method(app_id, method, data)
    return response.text()

if __name__ == "__main__":
    server.run()
```

## Adding Dapr Resources

Expose Dapr component metadata as MCP resources:

```python
@server.resource("dapr://components")
def list_components() -> str:
    """Lists all registered Dapr components."""
    import httpx
    response = httpx.get("http://localhost:3500/v1.0/metadata")
    return response.text

@server.resource("dapr://state/{key}")
def get_state_resource(key: str) -> str:
    """Retrieves state for a given key."""
    result = dapr_client.get_state("statestore", key)
    return result.data.decode() if result.data else ""
```

## Running the MCP Server with Dapr

```bash
dapr run --app-id dapr-mcp-server \
  --app-port 8080 \
  --dapr-http-port 3500 \
  --components-path ./components \
  -- python mcp_server.py
```

## Connecting Claude Desktop

Add the MCP server to Claude Desktop's configuration:

```json
{
  "mcpServers": {
    "dapr-tools": {
      "command": "python",
      "args": ["/path/to/mcp_server.py"],
      "env": {
        "DAPR_HTTP_PORT": "3500"
      }
    }
  }
}
```

## Using Dapr MCP Tools from an Agent

From a Dapr Agent, connect to MCP servers as tool providers:

```python
from dapr_agents import Agent
from dapr_agents.mcp import MCPToolProvider

class MCPEnabledAgent(Agent):
    name = "mcp-agent"
    instructions = "You can interact with Dapr services via MCP tools."

    def __init__(self):
        super().__init__()
        # Load tools from an MCP server
        mcp_tools = MCPToolProvider.from_server("dapr-tools")
        self.add_tools(mcp_tools)
```

## Security Considerations

Restrict which Dapr operations are exposed as MCP tools:

```python
# Only expose read operations for public-facing MCP servers
@server.tool("get_product")
def get_product(product_id: str) -> str:
    """Gets product information by ID (read-only)."""
    result = dapr_client.get_state("products-store", product_id)
    return result.data.decode()
```

Use Dapr access control policies to restrict what the MCP server can do internally.

## Summary

Dapr's MCP integration exposes Dapr state stores, pub/sub topics, and service invocation as MCP tools and resources. Build MCP servers using `DaprMCPServer`, connect Claude Desktop or any MCP client, and consume Dapr tools from Dapr Agents via `MCPToolProvider`. This makes Dapr's full component ecosystem available to any MCP-compatible AI application.
