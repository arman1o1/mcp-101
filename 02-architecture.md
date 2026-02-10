# Chapter 2: Core Architecture — Hosts, Clients & Servers

## Learning Objectives

By the end of this chapter, you will:

- Understand the three-layer architecture of MCP
- Know the responsibilities of Hosts, Clients, and Servers
- Understand how JSON-RPC 2.0 powers MCP communication
- See how MCP's design compares to the Language Server Protocol

---

## The Three-Layer Model

MCP follows a clean, layered architecture with three distinct components:

```mermaid
graph TB
    subgraph Host ["Host (e.g., Claude Desktop, VS Code)"]
        LLM["LLM / AI Model"]
        C1["MCP Client 1"]
        C2["MCP Client 2"]
        C3["MCP Client 3"]
    end

    S1["MCP Server 1<br/>(GitHub)"]
    S2["MCP Server 2<br/>(PostgreSQL)"]
    S3["MCP Server 3<br/>(Slack)"]

    LLM <--> C1
    LLM <--> C2
    LLM <--> C3
    C1 <-- "JSON-RPC 2.0" --> S1
    C2 <-- "JSON-RPC 2.0" --> S2
    C3 <-- "JSON-RPC 2.0" --> S3
```

Each layer has a specific role. Let's break them down.

---

## Hosts

A **Host** is the top-level application that the user interacts with. It manages the overall experience and orchestrates connections to one or more MCP servers.

### Examples of Hosts

- **Claude Desktop** — Anthropic's desktop AI assistant
- **VS Code / Cursor** — Code editors with AI features
- **Custom AI applications** — Your own chatbots, agents, or pipelines

### Host Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Lifecycle management** | Launches and shuts down MCP clients and servers |
| **Server discovery** | Reads configuration to know which servers to connect to |
| **Permission control** | Manages user consent for tool invocations and data access |
| **Orchestration** | Routes LLM requests to the appropriate client/server |
| **Security enforcement** | Applies trust boundaries, sandboxing, and access policies |

### Configuration Example

Hosts typically use a JSON configuration file to define which servers to connect to:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx..."
      }
    },
    "postgres": {
      "command": "python",
      "args": ["postgres_server.py"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost/mydb"
      }
    }
  }
}
```

The host reads this config, spawns each server process, and creates a dedicated client for each.

---

## Clients

A **Client** is an intermediary created by the host to maintain a **1:1 connection** with a single MCP server. The host creates one client per server.

### Client Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Connection management** | Establishes and maintains the transport connection |
| **Protocol translation** | Converts between the host's internal format and MCP JSON-RPC |
| **Session state** | Tracks negotiated capabilities and protocol version |
| **Request routing** | Forwards LLM tool calls to the server and returns results |
| **Capability tracking** | Remembers what the server supports (tools, resources, prompts) |

### Key Design Choice: 1:1 Mapping

Each client connects to exactly **one** server. This isolation is intentional:

- **Security**: A compromised server can't affect other connections
- **Reliability**: One server crashing doesn't break others
- **Simplicity**: Each client manages a single session lifecycle

```
Host
├── Client A ←→ Server A (GitHub)
├── Client B ←→ Server B (PostgreSQL)
└── Client C ←→ Server C (Slack)
```

---

## Servers

A **Server** is a program that exposes external capabilities to AI applications through the MCP protocol. Servers are the bridge between the AI world and real-world systems.

### Examples of Servers

| Server | What It Exposes |
|--------|----------------|
| **GitHub Server** | Repository files, issues, PRs as tools and resources |
| **PostgreSQL Server** | SQL query execution, schema inspection |
| **Slack Server** | Message sending, channel listing |
| **Filesystem Server** | File reading, writing, directory listing |
| **Weather Server** | Weather data lookups |

### Server Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Capability declaration** | Announces supported tools, resources, and prompts |
| **Request handling** | Processes `tools/call`, `resources/read`, `prompts/get` |
| **External translation** | Converts MCP requests into API calls, DB queries, etc. |
| **Response formatting** | Returns results in MCP's standard content format |
| **Access control** | Validates permissions and enforces security policies |

### What a Server Exposes

Servers declare their capabilities through three primitives:

```mermaid
graph LR
    S["MCP Server"]
    T["🔧 Tools<br/>(Actions)"]
    R["📄 Resources<br/>(Data)"]
    P["💬 Prompts<br/>(Templates)"]

    S --> T
    S --> R
    S --> P

    T --> T1["run_query()"]
    T --> T2["create_issue()"]
    R --> R1["file://config.yaml"]
    R --> R2["db://users/123"]
    P --> P1["code_review"]
    P --> P2["sql_explain"]
```

---

## The Communication Flow

Here's a complete flow showing how a user request travels through all three layers:

```mermaid
sequenceDiagram
    actor User
    participant Host
    participant Client
    participant Server
    participant External as External System

    User->>Host: "What's in my GitHub repo?"
    Host->>Host: LLM decides to use github tools
    Host->>Client: Forward tool call
    Client->>Server: tools/call("list_files", {repo: "my-repo"})
    Server->>External: GitHub API request
    External-->>Server: API response
    Server-->>Client: MCP result (file listing)
    Client-->>Host: Forward result to LLM
    Host->>Host: LLM generates response
    Host-->>User: "Your repo contains: main.py, README.md, ..."
```

### Step-by-Step Breakdown

1. **User** sends a natural language request to the Host
2. **Host** passes the request to the LLM along with available tool descriptions
3. **LLM** decides which tool to call and generates arguments
4. **Host** routes the tool call to the appropriate Client
5. **Client** sends a JSON-RPC `tools/call` request to the Server
6. **Server** translates the request into an external system call (API, DB, file system)
7. **External system** returns data
8. **Server** formats the result as MCP content and sends it back
9. **Client** forwards the result to the Host
10. **Host** provides the result to the LLM, which generates the final response

---

## JSON-RPC 2.0: The Message Format

All MCP communication uses **JSON-RPC 2.0**, a lightweight remote procedure call protocol. Every message is a JSON object with a specific structure.

### Request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "run_query",
    "arguments": {
      "sql": "SELECT * FROM users LIMIT 5"
    }
  }
}
```

### Response (Success)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "id | name | email\n1 | Alice | alice@example.com\n..."
      }
    ]
  }
}
```

### Response (Error)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid SQL: syntax error near 'SELCT'"
  }
}
```

### Notifications (No Response Expected)

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

> **Note**: Notifications have no `id` field. They are fire-and-forget messages — the sender doesn't expect a response.

### Why JSON-RPC 2.0?

| Property | Benefit |
|----------|---------|
| **Simple** | Just JSON — easy to implement in any language |
| **Bidirectional** | Both client and server can send requests |
| **Battle-tested** | Used widely in LSP, Ethereum, and other protocols |
| **Minimal overhead** | No headers, no HTTP semantics to deal with |
| **Notification support** | One-way messages for events and updates |

---

## Comparison with LSP

MCP was directly inspired by the **Language Server Protocol (LSP)**. The parallels are striking:

| Aspect | LSP | MCP |
|--------|-----|-----|
| **Problem** | N editors × M languages | N AI apps × M tools |
| **Solution** | Standard editor↔language protocol | Standard AI↔tool protocol |
| **Architecture** | Editor → Client → Language Server | Host → Client → MCP Server |
| **Message format** | JSON-RPC 2.0 | JSON-RPC 2.0 |
| **Discovery** | Capabilities negotiation | Capabilities negotiation |
| **Impact** | Every editor supports every language | Every AI app supports every tool |

The key difference: MCP adds **bidirectional AI-specific features** like sampling (server → LLM requests) and rich content types (images, structured data) that LSP doesn't need.

---

## Multiple Servers in Practice

Real-world setups often connect a host to many servers simultaneously:

```mermaid
graph TB
    subgraph Host ["Claude Desktop"]
        LLM["Claude 3.5"]
        C1["Client"]
        C2["Client"]
        C3["Client"]
        C4["Client"]
    end

    S1["📁 Filesystem<br/>Server"]
    S2["🐙 GitHub<br/>Server"]
    S3["🐘 PostgreSQL<br/>Server"]
    S4["💬 Slack<br/>Server"]

    LLM <--> C1 <--> S1
    LLM <--> C2 <--> S2
    LLM <--> C3 <--> S3
    LLM <--> C4 <--> S4
```

When the LLM needs to perform an action, the host determines which server has the relevant tool and routes the request through the correct client. The LLM sees all tools from all servers as a unified set.

---

## Summary

| Component | Role | Quantity |
|-----------|------|----------|
| **Host** | User-facing app, orchestrator | 1 per application |
| **Client** | Protocol connector | 1 per server connection |
| **Server** | External capability provider | Many per host |

Key takeaways:

- MCP uses a **Host → Client → Server** three-layer architecture
- **Hosts** manage the user experience and orchestrate connections
- **Clients** maintain 1:1 connections with servers (isolation by design)
- **Servers** expose tools, resources, and prompts to AI applications
- All communication uses **JSON-RPC 2.0** — simple, bidirectional JSON messages
- The design is inspired by **LSP**, adapted for AI-specific needs

---

## What's Next

In **Chapter 3**, we'll explore the protocol **lifecycle** — how connections are initialized, how capabilities are negotiated, and how sessions are managed.
