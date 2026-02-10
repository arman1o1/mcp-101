# Chapter 8: Building Your First MCP Server (Python)

## Learning Objectives

By the end of this chapter, you will:

- Set up a Python development environment for MCP
- Build a complete MCP server with tools, resources, and prompts
- Test your server with MCP Inspector
- Connect your server to Claude Desktop
- Follow production-ready patterns

---

## Prerequisites

- Python 3.10+
- Basic command-line knowledge
- Chapters 1-7 of this course

---

## Step 1: Project Setup

### Using `uv` (Recommended)

```bash
# Create project directory
mkdir weather-mcp-server
cd weather-mcp-server

# Initialize with uv
uv init

# Add the MCP SDK with CLI tools
uv add "mcp[cli]"
```

### Using pip

```bash
mkdir weather-mcp-server
cd weather-mcp-server

python -m venv .venv
# Windows:
.venv\Scripts\activate
# macOS/Linux:
# source .venv/bin/activate

pip install "mcp[cli]"
```

### Project Structure

```
weather-mcp-server/
├── pyproject.toml
├── server.py          # Main server file
└── README.md
```

---

## Step 2: Build the Server

We'll build a **Weather Information Server** that demonstrates all three MCP primitives.

### `server.py` — Complete Implementation

```python
"""
Weather MCP Server
==================
A demo MCP server that provides weather-related tools, resources, and prompts.
This uses mock data for demonstration — replace with real API calls in production.
"""

import json
from datetime import datetime

from mcp.server.mcpserver import MCPServer

# Create the MCP server
mcp = MCPServer(
    name="WeatherServer",
    instructions="This server provides weather information. "
    "Use the get_weather tool to check current conditions "
    "and the get_forecast tool for multi-day forecasts.",
    version="1.0.0",
)

# ──────────────────────────────────────────────────────
# Mock Data (replace with real API calls in production)
# ──────────────────────────────────────────────────────

WEATHER_DATA = {
    "london": {
        "city": "London",
        "temperature": 12,
        "condition": "Cloudy",
        "humidity": 78,
        "wind_speed": 15,
        "unit": "celsius",
    },
    "new york": {
        "city": "New York",
        "temperature": 22,
        "condition": "Sunny",
        "humidity": 45,
        "wind_speed": 8,
        "unit": "celsius",
    },
    "tokyo": {
        "city": "Tokyo",
        "temperature": 28,
        "condition": "Partly Cloudy",
        "humidity": 65,
        "wind_speed": 12,
        "unit": "celsius",
    },
    "mumbai": {
        "city": "Mumbai",
        "temperature": 33,
        "condition": "Hot & Humid",
        "humidity": 85,
        "wind_speed": 10,
        "unit": "celsius",
    },
    "sydney": {
        "city": "Sydney",
        "temperature": 19,
        "condition": "Clear",
        "humidity": 55,
        "wind_speed": 20,
        "unit": "celsius",
    },
}

SUPPORTED_CITIES = list(WEATHER_DATA.keys())

# ──────────────────────────────────────────────────────
# TOOLS
# ──────────────────────────────────────────────────────


@mcp.tool()
def get_weather(city: str) -> str:
    """Get current weather conditions for a city.

    Args:
        city: Name of the city (e.g., 'London', 'New York', 'Tokyo')
    """
    city_lower = city.lower()
    if city_lower not in WEATHER_DATA:
        available = ", ".join(WEATHER_DATA[c]["city"] for c in WEATHER_DATA)
        raise ValueError(
            f"City '{city}' not found. Available cities: {available}"
        )

    data = WEATHER_DATA[city_lower]
    return json.dumps(data, indent=2)


@mcp.tool()
def get_forecast(city: str, days: int = 3) -> str:
    """Get a multi-day weather forecast for a city.

    Args:
        city: Name of the city
        days: Number of forecast days (1-7, default: 3)
    """
    if days < 1 or days > 7:
        raise ValueError("Days must be between 1 and 7")

    city_lower = city.lower()
    if city_lower not in WEATHER_DATA:
        available = ", ".join(WEATHER_DATA[c]["city"] for c in WEATHER_DATA)
        raise ValueError(
            f"City '{city}' not found. Available cities: {available}"
        )

    base = WEATHER_DATA[city_lower]
    forecast = []
    conditions = ["Sunny", "Cloudy", "Partly Cloudy", "Rainy", "Clear"]

    for i in range(days):
        forecast.append({
            "day": i + 1,
            "temperature_high": base["temperature"] + 2 - i,
            "temperature_low": base["temperature"] - 5 + i,
            "condition": conditions[i % len(conditions)],
            "precipitation_chance": (i * 15 + 10) % 100,
        })

    result = {
        "city": base["city"],
        "forecast_days": days,
        "forecast": forecast,
    }
    return json.dumps(result, indent=2)


@mcp.tool()
def compare_weather(cities: str) -> str:
    """Compare weather between multiple cities.

    Args:
        cities: Comma-separated list of city names (e.g., 'London, Tokyo, Mumbai')
    """
    city_list = [c.strip().lower() for c in cities.split(",")]
    comparison = []

    for city in city_list:
        if city in WEATHER_DATA:
            data = WEATHER_DATA[city]
            comparison.append({
                "city": data["city"],
                "temperature": f"{data['temperature']}°C",
                "condition": data["condition"],
                "humidity": f"{data['humidity']}%",
            })
        else:
            comparison.append({
                "city": city.title(),
                "error": "City not found",
            })

    return json.dumps(comparison, indent=2)


# ──────────────────────────────────────────────────────
# RESOURCES
# ──────────────────────────────────────────────────────


@mcp.resource("weather://cities")
def list_supported_cities() -> str:
    """List all cities with available weather data."""
    cities = [WEATHER_DATA[c]["city"] for c in WEATHER_DATA]
    return json.dumps({"supported_cities": cities, "total": len(cities)})


@mcp.resource("weather://city/{city_name}")
def get_city_details(city_name: str) -> str:
    """Get detailed weather information for a specific city."""
    city_lower = city_name.lower()
    if city_lower not in WEATHER_DATA:
        return json.dumps({"error": f"City '{city_name}' not found"})

    data = WEATHER_DATA[city_lower].copy()
    data["retrieved_at"] = datetime.now().isoformat()
    return json.dumps(data, indent=2)


@mcp.resource("weather://about")
def about() -> str:
    """Information about this weather server."""
    return """# Weather MCP Server

This server provides weather information for select cities worldwide.

## Available Tools
- **get_weather**: Current conditions for a city
- **get_forecast**: Multi-day forecast
- **compare_weather**: Side-by-side comparison

## Available Cities
London, New York, Tokyo, Mumbai, Sydney

## Data Source
This demo uses mock data. In production, connect to a real weather API.
"""


# ──────────────────────────────────────────────────────
# PROMPTS
# ──────────────────────────────────────────────────────


@mcp.prompt()
def weather_report(city: str) -> str:
    """Generate a detailed weather report for a city."""
    return f"""Please provide a comprehensive weather report for {city}.

Use the get_weather tool to fetch current conditions, then format it as:

## Weather Report: {city}
**Date**: [current date]

### Current Conditions
- Temperature
- Sky conditions
- Humidity
- Wind speed

### What to Wear
Based on the conditions, suggest appropriate clothing.

### Travel Advisory
Any weather-related travel advice for visitors."""


@mcp.prompt()
def travel_comparison(cities: str) -> str:
    """Compare weather between cities for travel planning."""
    return f"""I'm planning a trip and want to compare the weather in these cities: {cities}

Use the compare_weather tool to get data for all cities, then:

1. Create a comparison table with temperature, conditions, and humidity
2. Rank them from best to worst travel weather
3. Suggest the best time to visit each
4. Recommend what to pack for each destination"""


@mcp.prompt()
def weather_briefing(city: str, days: str = "3") -> str:
    """Get a quick weather briefing with forecast."""
    return f"""Give me a quick weather briefing for {city}.

Steps:
1. Use get_weather to get current conditions
2. Use get_forecast for the next {days} days
3. Summarize in 2-3 sentences: current weather, trend direction, and any notable changes coming up"""


# ──────────────────────────────────────────────────────
# Run the server
# ──────────────────────────────────────────────────────

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

---

## Step 3: Test with MCP Inspector

The MCP SDK includes a built-in testing tool called **MCP Inspector**:

```bash
mcp dev server.py
```

This opens a web UI (usually at `http://localhost:5173`) where you can:

1. **Browse tools** — See all registered tools with their schemas
2. **Call tools** — Input test arguments and see results
3. **Browse resources** — List and read all resources
4. **Browse prompts** — List and get prompts with test arguments
5. **View raw messages** — See the actual JSON-RPC communication

### Testing Checklist

- [ ] `tools/list` shows all 3 tools (get_weather, get_forecast, compare_weather)
- [ ] `get_weather("London")` returns valid weather data
- [ ] `get_weather("InvalidCity")` returns a helpful error
- [ ] `get_forecast("Tokyo", 5)` returns 5-day forecast
- [ ] `get_forecast("London", 10)` returns validation error
- [ ] `compare_weather("London, Tokyo, Mumbai")` returns comparison
- [ ] `resources/list` shows all resources
- [ ] Reading `weather://cities` returns city list
- [ ] Reading `weather://city/london` returns London's weather
- [ ] `prompts/list` shows all prompts
- [ ] Getting `weather_report` with city argument returns formatted prompt

---

## Step 4: Connect to Claude Desktop

### Find Your Config File

| OS | Config Path |
|----|-------------|
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` |
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| **Linux** | `~/.config/Claude/claude_desktop_config.json` |

### Add Your Server

```json
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": [
        "--directory", "C:/path/to/weather-mcp-server",
        "run", "server.py"
      ]
    }
  }
}
```

Or with Python directly:

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["C:/path/to/weather-mcp-server/server.py"]
    }
  }
}
```

### Restart Claude Desktop

After saving the config, restart Claude Desktop. You should see a hammer icon (🔨) indicating tools are available. Try asking:

- *"What's the weather in Tokyo?"*
- *"Compare weather between London and Mumbai"*
- *"Give me a 5-day forecast for Sydney"*

---

## Step 5: Run with Streamable HTTP

For remote access, switch to Streamable HTTP:

```python
if __name__ == "__main__":
    import sys

    transport = sys.argv[1] if len(sys.argv) > 1 else "stdio"

    if transport == "http":
        mcp.run(
            transport="streamable-http",
            host="0.0.0.0",
            port=8000,
            json_response=True,
        )
    else:
        mcp.run(transport="stdio")
```

Run:

```bash
python server.py http
```

The server is now accessible at `http://localhost:8000/mcp`.

---

## Code Walkthrough

Let's break down the key patterns used in this server:

### Pattern 1: MCPServer Initialization

```python
mcp = MCPServer(
    name="WeatherServer",                    # Server identity
    instructions="Use get_weather for...",   # Guidance for the LLM
    version="1.0.0",                         # Server version
)
```

The `instructions` field is sent to the LLM during initialization — it helps the model understand how to use your server effectively.

### Pattern 2: Error Handling with Exceptions

```python
@mcp.tool()
def get_forecast(city: str, days: int = 3) -> str:
    if days < 1 or days > 7:
        raise ValueError("Days must be between 1 and 7")
    # ...
```

Raised exceptions are automatically caught by the SDK and returned as tool errors (`isError: true`). The LLM sees the error message and can adapt.

### Pattern 3: Type-Hinted Arguments

```python
def get_forecast(city: str, days: int = 3) -> str:
```

- `city: str` → Required string argument
- `days: int = 3` → Optional integer with default value
- `-> str` → Return type (converted to text content)

The SDK auto-generates JSON Schema from these type hints.

### Pattern 4: Dynamic Resources with Templates

```python
@mcp.resource("weather://city/{city_name}")
def get_city_details(city_name: str) -> str:
```

The `{city_name}` in the URI is a template variable. When a client reads `weather://city/london`, the SDK extracts `city_name = "london"` and calls the function.

---

## Exercise: Extend the Weather Server

Add these features to the weather server:

1. **New Tool: `convert_temperature(value, from_unit, to_unit)`**
   - Convert between Celsius, Fahrenheit, and Kelvin

2. **New Resource: `weather://alerts/{city_name}`**
   - Return weather alerts for a city (mock data is fine)

3. **New Prompt: `weekly_planner(city)`**
   - Generate a week-long activity plan based on the forecast

4. **Switch transport** based on a command-line argument:
   - `python server.py` → stdio
   - `python server.py --http` → Streamable HTTP on port 8000

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Forgetting `if __name__ == "__main__"` | Always guard the `mcp.run()` call |
| Print statements breaking stdio | Use `stderr` for logging, never `stdout` |
| Missing type hints | Always annotate every parameter and return type |
| Vague tool descriptions | Write descriptions as if explaining to the LLM |
| Not handling errors | Use `raise ValueError(...)` for user-facing errors |
| Hardcoded paths | Use `pathlib.Path` or environment variables |

---

## Summary

- Set up a project with `uv init` + `uv add "mcp[cli]"`
- Create `MCPServer` with name, instructions, and version
- Register tools with `@mcp.tool()`, resources with `@mcp.resource()`, prompts with `@mcp.prompt()`
- Test with `mcp dev server.py` (MCP Inspector)
- Connect to Claude Desktop via `claude_desktop_config.json`
- Run remotely with `transport="streamable-http"`
- Handle errors by raising exceptions
- Use type hints for automatic schema generation

---

## What's Next

In **Chapter 9**, we'll build the other side — an **MCP Client** that connects to your server programmatically, without needing Claude Desktop.
