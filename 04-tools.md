# Chapter 4: Tools — Giving LLMs the Power to Act

## Learning Objectives

By the end of this chapter, you will:

- Understand what MCP tools are and how they differ from resources and prompts
- Know the complete tool lifecycle: discovery → invocation → result
- Read and write tool definitions with JSON Schema
- Build tools in Python using the `@mcp.tool()` decorator
- Handle errors and structured output in tools

---

## What Are Tools?

**Tools** are the most powerful MCP primitive. They represent **executable functions** that an LLM can invoke to perform actions in external systems.

| Property | Description |
| :--- | :--- |
| **Controlled by** | The AI model (the LLM decides when and how to call them) |
| **Purpose** | Execute actions, compute results, interact with external systems |
| **Direction** | Client → Server (client invokes, server executes) |
| **Examples** | Run SQL queries, send emails, create files, call APIs |

> **Key principle**: Tools are **model-controlled**. The LLM sees tool descriptions and decides autonomously which tools to call and with what arguments. The host typically asks for user confirmation before executing.

### Tools vs Resources vs Prompts

| | Tools | Resources | Prompts |
| :--- | :--- | :--- | :--- |
| **Who decides** | Model | Application | User |
| **Operation** | Execute actions | Read data | Get templates |
| **Side effects** | Yes (mutations) | No (read-only) | No |
| **Analogy** | Functions | Files | Templates |

---

## Tool Discovery: `tools/list`

Before using tools, the client asks the server what's available:

### List Tools Request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}
```

### Response

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "add",
        "description": "Add two numbers together",
        "inputSchema": {
          "type": "object",
          "properties": {
            "a": {
              "type": "integer",
              "description": "First number"
            },
            "b": {
              "type": "integer",
              "description": "Second number"
            }
          },
          "required": ["a", "b"]
        }
      },
      {
        "name": "run_query",
        "description": "Execute a SQL query against the database",
        "inputSchema": {
          "type": "object",
          "properties": {
            "sql": {
              "type": "string",
              "description": "The SQL query to execute"
            }
          },
          "required": ["sql"]
        }
      }
    ]
  }
}
```

### Tool Definition Fields

| Field | Required | Description |
| :--- | :--- | :--- |
| `name` | Yes | Unique identifier for the tool |
| `description` | Yes | Human-readable description (shown to the LLM) |
| `inputSchema` | Yes | JSON Schema defining the expected arguments |

> **Important**: The `description` is critical — it's what the LLM reads to decide whether to use the tool. Write clear, specific descriptions that explain when and how to use the tool.

---

## Tool Invocation: `tools/call`

When the LLM decides to use a tool, the client sends a `tools/call` request:

### Call Tool Request

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "add",
    "arguments": {
      "a": 15,
      "b": 27
    }
  }
}
```

### Successful Response

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "42"
      }
    ],
    "isError": false
  }
}
```

### Content Types

Tool results can contain different content types:

| Type | Usage |
| :--- | :--- |
| `text` | Plain text results |
| `image` | Base64-encoded images |
| `resource` | Embedded resource references |

```json
{
  "content": [
    {
      "type": "text",
      "text": "Query returned 5 rows"
    },
    {
      "type": "image",
      "data": "iVBORw0KGgoAAAA...",
      "mimeType": "image/png"
    }
  ]
}
```

### Structured Output

Tools can also return **structured content** alongside text, enabling machine-readable results:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "15 + 27 = 42"
      }
    ],
    "structuredContent": {
      "result": 42,
      "operation": "addition"
    }
  }
}
```

---

## Error Handling in Tools

Tools can fail. MCP provides two levels of error handling:

### Tool-Level Errors (Expected Failures)

When a tool executes but produces an error result (e.g., invalid SQL), set `isError: true`:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Error: relation 'users' does not exist"
      }
    ],
    "isError": true
  }
}
```

The LLM sees this error and can decide to retry, use a different approach, or report it to the user. This is **not** a protocol error — the tool executed, it just had a problem.

### Protocol-Level Errors (Unexpected Failures)

When something goes wrong at the protocol level (tool not found, invalid request), use a JSON-RPC error:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32602,
    "message": "Unknown tool: 'nonexistent_tool'"
  }
}
```

### Error Handling Summary

| Scenario | Mechanism | LLM Sees It? |
| :--- | :--- | :--- |
| Tool ran but failed (bad input, etc.) | `isError: true` in result | Yes — can adapt |
| Tool not found | JSON-RPC error | Depends on host |
| Invalid arguments | JSON-RPC error | Depends on host |
| Server crash | Transport error | No — host handles |

---

## Dynamic Tool Lists: `listChanged`

If a server declares `"tools": { "listChanged": true }` in its capabilities, it can notify the client when its tool list changes:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

Upon receiving this notification, the client should call `tools/list` again to get the updated list. This enables **dynamic servers** that add or remove tools based on context.

---

## Building Tools in Python

Now let's build tools using the official MCP Python SDK. The SDK uses a decorator-based API that makes tool creation simple.

### Setup

```bash
# Create a new project
mkdir my-mcp-server && cd my-mcp-server

# Install the MCP SDK
uv init
uv add "mcp[cli]"
```

### Basic Tool Server

```python
from mcp.server.mcpserver import MCPServer

# Create the server
mcp = MCPServer("Calculator")


# Define a tool using the @tool decorator
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers together."""
    return a + b


@mcp.tool()
def multiply(a: float, b: float) -> float:
    """Multiply two numbers together."""
    return a * b


@mcp.tool()
def divide(a: float, b: float) -> float:
    """Divide a by b. Returns an error if b is zero."""
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

**Key points:**

- `@mcp.tool()` registers the function as an MCP tool
- The **docstring** becomes the tool's `description`
- **Type hints** generate the `inputSchema` automatically
- **Return values** are converted to text content
- **Exceptions** are caught and returned as tool errors (`isError: true`)

### Structured Output with Pydantic

Tools can return structured data (JSON) instead of just text strings. The SDK supports `pydantic.BaseModel`, `dict`, and `dataclasses`.

```python
from pydantic import BaseModel, Field

class WeatherData(BaseModel):
    temperature: float = Field(description="Temperature in Celsius")
    conditions: str = Field(description="Weather conditions")
    humidity: float

@mcp.tool()
def get_weather_data(city: str) -> WeatherData:
    """Get structured weather data."""
    return WeatherData(
        temperature=22.5,
        conditions="Sunny",
        humidity=45.0
    )
```

The LLM will receive this as a structured JSON object, making it easier to parse programmatically.

### How the SDK Maps Python to MCP

```text
Python                          MCP Tool Definition
──────                          ───────────────────
Function name          →        tool name
Docstring              →        description
Type hints             →        inputSchema (JSON Schema)
Return value           →        content[0].text
Raised exception       →        content[0].text + isError: true
```

### Tool with Complex Arguments

```python
from typing import Optional


@mcp.tool()
def search_users(
    query: str,
    limit: int = 10,
    include_inactive: bool = False,
    department: Optional[str] = None,
) -> str:
    """Search for users in the database.

    Args:
        query: Search term to match against user names or emails
        limit: Maximum number of results to return (default: 10)
        include_inactive: Whether to include inactive users
        department: Optional department filter
    """
    # Build and execute query...
    results = [
        {"name": "Alice", "email": "alice@example.com"},
        {"name": "Bob", "email": "bob@example.com"},
    ]
    return str(results)
```

This generates the following JSON Schema:

```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "Search term to match against user names or emails"
    },
    "limit": {
      "type": "integer",
      "description": "Maximum number of results to return (default: 10)",
      "default": 10
    },
    "include_inactive": {
      "type": "boolean",
      "description": "Whether to include inactive users",
      "default": false
    },
    "department": {
      "type": "string",
      "description": "Optional department filter"
    }
  },
  "required": ["query"]
}
```

### Async Tools

Tools can be asynchronous for I/O-bound operations:

```python
import httpx


@mcp.tool()
async def fetch_weather(city: str) -> str:
    """Get current weather for a city."""
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.weather.example.com/current?city={city}"
        )
        data = response.json()
        return f"Weather in {city}: {data['temperature']}°C, {data['condition']}"
```

---

## Exercise: Build a Calculator Server

Build a complete MCP calculator server with the following tools:

1. **`add(a, b)`** — Add two numbers
2. **`subtract(a, b)`** — Subtract b from a
3. **`multiply(a, b)`** — Multiply two numbers
4. **`divide(a, b)`** — Divide a by b (handle division by zero)
5. **`power(base, exponent)`** — Calculate base^exponent
6. **`sqrt(n)`** — Calculate square root (handle negative numbers)

**Requirements:**

- Use proper type hints (`int` or `float`)
- Write descriptive docstrings
- Handle edge cases with meaningful error messages
- Run with stdio transport

**Test with MCP Inspector:**

```bash
mcp dev calculator_server.py
```

This opens a web UI where you can browse your tools, call them with test arguments, and inspect the results.

---

## Best Practices for Tool Design

| Practice | Why |
| :--- | :--- |
| **Write clear descriptions** | The LLM uses these to decide when to use the tool |
| **Use specific type hints** | Generates accurate JSON Schema for validation |
| **Handle errors gracefully** | Raise exceptions with descriptive messages |
| **Keep tools focused** | One tool = one action (avoid "god tools") |
| **Return useful text** | The LLM reads the text result — make it informative |
| **Use structured output** | For machine-readable results alongside text |
| **Document arguments** | Include arg descriptions in the docstring |

---

## Summary

- **Tools** are model-controlled executable functions — the most powerful MCP primitive
- Discovery via `tools/list` returns tool names, descriptions, and JSON Schema
- Invocation via `tools/call` sends arguments and receives content results
- Results can contain **text**, **images**, and **structured data**
- Errors are handled at **tool level** (`isError: true`) and **protocol level** (JSON-RPC errors)
- In Python, use `@mcp.tool()` decorator — type hints auto-generate schemas
- Tools can be **sync** or **async**
- Dynamic tool lists use `listChanged` notifications

---

## What's Next

In **Chapter 5**, we'll explore **Resources** — how servers expose read-only data to LLMs for context injection.
