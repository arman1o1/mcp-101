# MCP-101: Mastering the Model Context Protocol

> A comprehensive, 14-chapter course on building MCP servers and clients — from first principles to production deployment.

---

## 📋 Prerequisites

- **Python 3.10+** installed
- **Basic Python knowledge** (functions, classes, async/await)
- **Basic LLM familiarity** (what LLMs are, how they generate text)
- No prior MCP experience required

## 🛠 Setup

```bash
# Install uv (recommended package manager)
pip install uv

# Install the MCP SDK
uv init my-mcp-project
cd my-mcp-project
uv add "mcp[cli]"
```

> [!NOTE]
> **FastMCP vs MCPServer**: You may see older tutorials reference a class called `FastMCP`. This has been renamed to `MCPServer` in the official SDK. The API is similar, but `MCPServer` is the current standard.

---

## 📚 Table of Contents

### Part 1: Foundations

| # | Chapter | Topics |
| :--- | :--- | :--- |
| 01 | [What is MCP? — The Big Picture](01-what-is-mcp.md) | Why MCP exists, the N×M problem, MCP vs REST vs function calling |
| 02 | [Core Architecture](02-architecture.md) | Hosts, Clients, Servers, JSON-RPC 2.0 |
| 03 | [Protocol Lifecycle](03-lifecycle.md) | Initialization handshake, capability negotiation, shutdown |

### Part 2: The Three Primitives

| # | Chapter | Topics |
| :--- | :--- | :--- |
| 04 | [Tools](04-tools.md) | Discovery, invocation, JSON Schema, structured output, error handling |
| 05 | [Resources](05-resources.md) | URIs, static vs dynamic, MIME types, subscriptions |
| 06 | [Prompts](06-prompts.md) | Templates, multi-turn messages, embedded resources, control hierarchy |

### Part 3: Infrastructure

| # | Chapter | Topics |
| :--- | :--- | :--- |
| 07 | [Transport Mechanisms](07-transports.md) | stdio, SSE, Streamable HTTP, choosing the right transport |

### Part 4: Hands-On Building

| # | Chapter | Topics |
| :--- | :--- | :--- |
| 08 | [Building Your First Server](08-building-server.md) | Complete weather server with tools, resources, prompts |
| 09 | [Building an MCP Client](09-building-client.md) | stdio & HTTP clients, interactive CLI, tool calling |

### Part 5: Advanced Topics

| # | Chapter | Topics |
| :--- | :--- | :--- |
| 10 | [Sampling & Elicitation](10-sampling-elicitation.md) | Server→LLM requests, user input, human-in-the-loop |
| 11 | [Security & Best Practices](11-security.md) | Threat model, authentication, input validation, audit logging |

### Part 6: Production

| # | Chapter | Topics |
| :--- | :--- | :--- |
| 12 | [Ecosystem & Real-World Servers](12-ecosystem.md) | Popular servers, evaluation criteria, industry adoption |
| 13 | [Multi-Server Orchestration](13-orchestration.md) | Tool routing, agentic workflows, design patterns |
| 14 | [Deploying to Production](14-deployment.md) | Docker, scaling, monitoring, CI/CD |

---

## 🎯 How to Use This Course

1. **Read sequentially** — Chapters build on each other
2. **Run the code** — Every code example is designed to work
3. **Do the exercises** — Hands-on practice solidifies understanding
4. **Test with MCP Inspector** — Use `mcp dev server.py` to test your servers
5. **Build something real** — Apply what you learn to a project you care about

## 📖 Key Resources

| Resource | Link |
| :--- | :--- |
| MCP Specification | [modelcontextprotocol.io](https://modelcontextprotocol.io) |
| Python SDK | [github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) |
| MCP Inspector | Included with `mcp[cli]` — run `mcp dev server.py` |
| Server Registry | [modelcontextprotocol.io/servers](https://modelcontextprotocol.io) |

---

*Built with ❤️ for the MCP community*
