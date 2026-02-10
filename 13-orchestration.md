# Chapter 13: Multi-Server Orchestration & Agentic Workflows

## Learning Objectives

By the end of this chapter, you will:

- Configure multiple MCP servers in a single host
- Understand tool routing and deduplication
- Design agentic workflows spanning multiple servers
- Build a multi-server configuration from scratch

---

## Why Multiple Servers?

Real-world AI workflows rarely need just one external system. A typical scenario:

```
"Read the latest sales data from PostgreSQL,
 generate a report, and post it to Slack"
```

This requires **three servers** working together:

```mermaid
graph TB
    subgraph Host ["AI Host"]
        LLM["LLM"]
        C1["Client 1"]
        C2["Client 2"]
        C3["Client 3"]
    end

    S1["🐘 PostgreSQL Server"]
    S2["📁 Filesystem Server"]
    S3["💬 Slack Server"]

    LLM <--> C1 <--> S1
    LLM <--> C2 <--> S2
    LLM <--> C3 <--> S3
```

The LLM sees a **unified set of tools** from all servers and orchestrates them automatically.

---

## Multi-Server Configuration

### Claude Desktop Configuration

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx"
      }
    },
    "postgres": {
      "command": "python",
      "args": ["servers/postgres_server.py"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost/mydb"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y", "@modelcontextprotocol/server-filesystem",
        "/home/user/projects"
      ]
    },
    "slack": {
      "command": "python",
      "args": ["servers/slack_server.py"],
      "env": {
        "SLACK_BOT_TOKEN": "xoxb-xxx"
      }
    }
  }
}
```

### What Happens at Startup

```mermaid
sequenceDiagram
    participant H as Host
    participant C1 as Client (GitHub)
    participant C2 as Client (Postgres)
    participant C3 as Client (FS)
    participant C4 as Client (Slack)

    H->>C1: Initialize GitHub Server
    H->>C2: Initialize Postgres Server
    H->>C3: Initialize Filesystem Server
    H->>C4: Initialize Slack Server

    par Parallel initialization
        C1-->>H: Tools: [create_issue, search_code, ...]
        C2-->>H: Tools: [run_query, list_tables, ...]
        C3-->>H: Tools: [read_file, write_file, ...]
        C4-->>H: Tools: [post_message, list_channels, ...]
    end

    Note over H: Host merges all tools into<br/>a unified set for the LLM
```

The host:

1. Launches each server as a separate process
2. Creates a dedicated client for each
3. Initializes all connections (capability negotiation)
4. Merges all tools, resources, and prompts into a unified catalog
5. Presents the combined set to the LLM

---

## Tool Routing

When the LLM calls a tool, the host must route it to the correct server:

```mermaid
graph LR
    LLM["LLM calls: run_query(...)"]
    H["Host Router"]
    C1["Client → GitHub"]
    C2["Client → Postgres"]
    C3["Client → Slack"]

    LLM --> H
    H -->|"run_query belongs to Postgres"| C2
    H -.->|"not here"| C1
    H -.->|"not here"| C3
```

The host maintains a mapping: **tool name → server/client**. When a tool call comes from the LLM, the host looks up which server registered it and forwards the request.

---

## Handling Tool Name Collisions

What happens when two servers register tools with the same name?

```
GitHub Server:  tool "search" → Search GitHub repositories
Slack Server:   tool "search" → Search Slack messages
```

### Strategies

| Strategy | Description | Pros | Cons |
|----------|-------------|------|------|
| **Namespacing** | Prefix with server name: `github.search`, `slack.search` | Clear, no ambiguity | Longer tool names |
| **Priority** | First registered wins | Simple | Loses functionality |
| **User choice** | Ask user which to use | Flexible | Interrupts flow |
| **LLM context** | Include server info in description | No name changes | May confuse LLM |

Most hosts use **namespacing** or **priority-based** resolution.

---

## Agentic Workflows

An **agentic workflow** is a multi-step task where the LLM autonomously chains tools from different servers to achieve a goal.

### Example: Daily Standup Report

```mermaid
sequenceDiagram
    actor User
    participant LLM
    participant GH as GitHub Server
    participant PG as Postgres Server
    participant SL as Slack Server

    User->>LLM: "Generate daily standup report<br/>and post to #engineering"

    Note over LLM: Step 1: Get recent commits
    LLM->>GH: tools/call("list_commits", {since: "yesterday"})
    GH-->>LLM: [commit1, commit2, commit3]

    Note over LLM: Step 2: Get open issues
    LLM->>GH: tools/call("list_issues", {state: "open"})
    GH-->>LLM: [issue1, issue2]

    Note over LLM: Step 3: Get deployment status
    LLM->>PG: tools/call("run_query", {sql: "SELECT * FROM deployments..."})
    PG-->>LLM: [deploy1, deploy2]

    Note over LLM: Step 4: Generate and post report
    LLM->>LLM: Compose standup report
    LLM->>SL: tools/call("post_message", {channel: "#engineering", text: "..."})
    SL-->>LLM: Message posted

    LLM-->>User: "✅ Standup report posted to #engineering"
```

### Example: Bug Investigation Workflow

```
User: "Investigate the increase in 500 errors from last night"

LLM's autonomous steps:
1. [Postgres] Query error logs: run_query("SELECT * FROM errors WHERE timestamp > ...")
2. [GitHub]   Find recent commits: list_commits({since: "yesterday"})
3. [GitHub]   Get changed files: get_commit_diff({sha: "abc123"})
4. [Filesystem] Read the changed code: read_file("/src/handler.py")
5. [GitHub]   Check related issues: search_issues("500 error")
6. LLM synthesizes all data into a root cause analysis
7. [Slack]    Post findings: post_message({channel: "#incidents", text: "..."})
```

The LLM decides the order, which tools to use, and how to combine results — all autonomously.

---

## Building a Multi-Server Client

Here's how to connect to multiple servers programmatically:

```python
"""
Multi-server MCP client that connects to multiple servers
and provides a unified interface.
"""

import asyncio
from contextlib import AsyncExitStack
from dataclasses import dataclass

from mcp import ClientSession, StdioServerParameters, types
from mcp.client.stdio import stdio_client


@dataclass
class ServerConnection:
    name: str
    session: ClientSession
    tools: list


class MultiServerClient:
    def __init__(self):
        self.connections: dict[str, ServerConnection] = {}
        self.tool_map: dict[str, str] = {}  # tool_name -> server_name
        self._exit_stack = AsyncExitStack()

    async def connect(self, name: str, command: str, args: list[str]):
        """Connect to a server and register its tools."""
        params = StdioServerParameters(command=command, args=args)

        # Use AsyncExitStack to properly manage context manager lifetimes
        read, write = await self._exit_stack.enter_async_context(
            stdio_client(params)
        )
        session = await self._exit_stack.enter_async_context(
            ClientSession(read, write)
        )
        await session.initialize()

        # Get tools
        tools_result = await session.list_tools()
        tool_names = [t.name for t in tools_result.tools]

        # Register
        self.connections[name] = ServerConnection(
            name=name, session=session, tools=tool_names
        )

        # Map tools to server (handle collisions with namespacing)
        for tool_name in tool_names:
            if tool_name in self.tool_map:
                # Collision! Use namespaced version
                existing = self.tool_map[tool_name]
                print(f"⚠️  Tool '{tool_name}' collision: "
                      f"{existing} vs {name}")
            self.tool_map[tool_name] = name

        print(f"✅ Connected to {name}: {tool_names}")

    async def call_tool(self, tool_name: str, arguments: dict) -> str:
        """Route a tool call to the correct server."""
        server_name = self.tool_map.get(tool_name)
        if not server_name:
            raise ValueError(f"Unknown tool: {tool_name}")

        conn = self.connections[server_name]
        result = await conn.session.call_tool(tool_name, arguments=arguments)

        texts = []
        for content in result.content:
            if isinstance(content, types.TextContent):
                texts.append(content.text)
        return "\n".join(texts)

    def list_all_tools(self) -> dict[str, list[str]]:
        """List all tools grouped by server."""
        return {
            name: conn.tools
            for name, conn in self.connections.items()
        }

    async def disconnect_all(self):
        """Cleanly shut down all server connections."""
        await self._exit_stack.aclose()
        self.connections.clear()
        self.tool_map.clear()


async def main():
    client = MultiServerClient()

    try:
        # Connect to multiple servers
        await client.connect("weather", "python", ["weather_server.py"])
        await client.connect("calculator", "python", ["calc_server.py"])

        # List all available tools
        print("\nAll tools:")
        for server, tools in client.list_all_tools().items():
            print(f"  {server}: {tools}")

        # Call tools from different servers
        weather = await client.call_tool("get_weather", {"city": "London"})
        print(f"\nWeather: {weather}")

        calc = await client.call_tool("add", {"a": 5, "b": 3})
        print(f"Calc: {calc}")
    finally:
        await client.disconnect_all()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Design Patterns for Multi-Server Workflows

### Pattern 1: Pipeline

Data flows linearly through servers:

```
Server A → Server B → Server C → Result
(fetch)    (process)   (deliver)
```

### Pattern 2: Fan-Out / Fan-In

Gather data from multiple servers, then combine:

```
        → Server A (data) ─┐
Query ──→ Server B (data) ──→ LLM combines → Result
        → Server C (data) ─┘
```

### Pattern 3: Conditional Routing

Choose servers based on intermediate results:

```
Server A → LLM decision → Server B or Server C
```

### Pattern 4: Iterative Refinement

Loop through servers until a condition is met:

```
while not satisfied:
    result = Server A → LLM evaluate → Server B (adjust)
```

---

## Best Practices

| Practice | Description |
|----------|-------------|
| **Minimize servers** | Only connect what you need — each server adds overhead |
| **Isolate secrets** | Each server gets only its own credentials |
| **Handle failures** | One server crashing shouldn't break others |
| **Log tool routes** | Track which server handled which tool call |
| **Test individually** | Test each server alone before combining |
| **Document combos** | Document which server combinations work well together |

---

## Summary

- Hosts can connect to **multiple MCP servers** simultaneously
- Each server gets its own **dedicated client** (1:1 isolation)
- The host **merges tools** from all servers into a unified catalog
- **Tool collisions** are handled via namespacing or priority
- **Agentic workflows** chain tools across servers autonomously
- Multi-server configurations use **JSON config files**
- Design patterns: **pipeline**, **fan-out/fan-in**, **conditional**, **iterative**

---

## What's Next

In **Chapter 14**, we'll cover **Deploying MCP Servers to Production** — packaging, containerization, monitoring, and scaling.
