# 🔗 MCP + Multi-Agent Systems: Complete Master Guide
## Model Context Protocol, Agent Communication, Coordination, and Distributed AI Architectures

> **Level:** Senior-level understanding (suitable for interviews and production systems)  
> **Scope:** Everything from MCP fundamentals through complex multi-agent orchestration  
> **Philosophy:** Understanding MCP unlocks the ability to build composable, scalable AI systems

---

## 📋 Table of Contents

### Part 1: MCP Fundamentals
- [1.1: What Is MCP?](#11-what-is-mcp)
- [1.2: The Problem MCP Solves](#12-the-problem-mcp-solves)
- [1.3: MCP Architecture Overview](#13-mcp-architecture-overview)
- [1.4: MCP vs Other Approaches](#14-mcp-vs-other-approaches)

### Part 2: Building MCP Servers
- [2.1: MCP Server Basics](#21-mcp-server-basics)
- [2.2: Implementing Tools via MCP](#22-implementing-tools-via-mcp)
- [2.3: Resource Management](#23-resource-management)
- [2.4: Prompts & Templates](#24-prompts--templates)

### Part 3: MCP Clients & Integration
- [3.1: MCP Client Patterns](#31-mcp-client-patterns)
- [3.2: Claude + MCP Integration](#32-claude--mcp-integration)
- [3.3: Handling Errors & Timeouts](#33-handling-errors--timeouts)
- [3.4: Security Considerations](#34-security-considerations)

### Part 4: Multi-Agent Fundamentals
- [4.1: What Are Multi-Agent Systems?](#41-what-are-multi-agent-systems)
- [4.2: Agent Types & Roles](#42-agent-types--roles)
- [4.3: Communication Patterns](#43-communication-patterns)
- [4.4: Coordination Mechanisms](#44-coordination-mechanisms)

### Part 5: Multi-Agent Architectures
- [5.1: Manager-Worker Pattern](#51-manager-worker-pattern)
- [5.2: Hierarchical Agents](#52-hierarchical-agents)
- [5.3: Peer-to-Peer Collaboration](#53-peer-to-peer-collaboration)
- [5.4: Specialized Agent Teams](#54-specialized-agent-teams)

### Part 6: Production Patterns
- [6.1: Scalability & Performance](#61-scalability--performance)
- [6.2: Reliability & Fault Tolerance](#62-reliability--fault-tolerance)
- [6.3: Monitoring & Observability](#63-monitoring--observability)
- [6.4: Cost Optimization](#64-cost-optimization)

### Part 7: Interview Q&A
- [7.1: Core Concept Questions](#71-core-concept-questions)
- [7.2: Implementation Questions](#72-implementation-questions)
- [7.3: Architecture Questions](#73-architecture-questions)

---

# Part 1: MCP Fundamentals

## 1.1 What Is MCP?

### The Simple Definition

**MCP = Model Context Protocol**

It's a standardized specification for how LLMs and tools/services communicate.

Think of it as: **A universal adapter for connecting LLMs to external resources.**

### The Real-World Analogy

```
Without MCP:
Company A builds tools in Python
Company B builds tools in Node.js
Company C builds tools in Go

Each company writes custom code to integrate with Claude:
Claude ← Custom adapter A → Tools (Python)
Claude ← Custom adapter B → Tools (Node.js)
Claude ← Custom adapter C → Tools (Go)

Problem: Too many custom adapters, each different, hard to maintain


With MCP:
Company A, B, C all build their tools following MCP spec
Claude ← Standard MCP client → MCP Server A (Python)
                              ↓
                           Tools (Python)

Claude ← Standard MCP client → MCP Server B (Node.js)
                              ↓
                           Tools (Node.js)

Claude ← Standard MCP client → MCP Server C (Go)
                              ↓
                           Tools (Go)

Benefit: One standard protocol. Companies follow the spec. Claude uses the same client for all.
```

### Key Insight: Composability

```
Without MCP:
"I want Claude + database + web API + file system"
→ Must write custom code to glue all these together
→ Tightly coupled, hard to swap components

With MCP:
"I want Claude + database + web API + file system"
→ Launch MCP server for database (someone else built it)
→ Launch MCP server for web API (someone else built it)
→ Launch MCP server for files (someone else built it)
→ Connect all via MCP protocol
→ Loosely coupled, easy to swap, combine, reuse
```

---

## 1.2 The Problem MCP Solves

### The Problem: Tool Integration Fragmentation

```
Scenario: You want to build an AI assistant that can:
- Query your database
- Call your APIs
- Read your files
- Send emails
- Access external services

Traditional approach:
1. For each tool, learn its API
2. Write code to call it
3. Parse the response
4. Format it for Claude
5. Repeat for the next tool

Result: 5 custom tool integrations = 5 different patterns
This scales poorly. With 50 tools, you have spaghetti code.
```

### The Solution: Standardized Protocol

```
MCP approach:
1. Each tool provider exposes MCP server (following the spec)
2. Claude knows how to talk to ANY MCP server
3. You just point Claude at the servers
4. Claude automatically discovers available tools
5. Everything works the same way

Result: 50 tools = 1 standard pattern repeated
Scalable, maintainable, composable.
```

### Why This Matters

```
Benefit 1: Reusability
You build an MCP server for your database
Anyone can use it (in their projects, with their LLMs)

Benefit 2: Modularity
Swap out database MCP server for another without changing Claude integration

Benefit 3: Standardization
Instead of learning 50 different APIs, you learn MCP once

Benefit 4: Ecosystem
Companies can release MCP servers publicly
Others build on them without custom integration
```

---

## 1.3 MCP Architecture Overview

### The Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ TIER 1: LLM Layer                                           │
│ (Claude, other language models)                             │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ MCP Protocol
                 │ (JSON-RPC 2.0 over stdio/network)
                 │
┌────────────────▼────────────────────────────────────────────┐
│ TIER 2: MCP Client                                          │
│ (Understands MCP protocol, routes requests/responses)       │
└────────────────┬────────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┬──────────────┐
    │            │            │              │
    ▼            ▼            ▼              ▼
┌────────┐  ┌────────┐  ┌────────┐    ┌──────────┐
│ MCP    │  │ MCP    │  │ MCP    │    │ MCP      │
│Server A│  │Server B│  │Server C│    │Server... │
│(Python)│  │(Node)  │  │(Go)    │    │          │
└───┬────┘  └───┬────┘  └───┬────┘    └──┬───────┘
    │           │           │             │
    ▼           ▼           ▼             ▼
[Database]  [APIs]     [Files]       [Services...]
```

### The Information Flow

```
Step 1: LLM needs external info
Claude: "I need to check user data in the database"

Step 2: Claude describes what it needs
Claude: "Query the users table where id=123"

Step 3: MCP Client translates request
MCP Client: Sends to appropriate MCP Server (database)
Format: JSON-RPC 2.0
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": {"query": "SELECT * FROM users WHERE id=123"}
  }
}

Step 4: MCP Server executes tool
MCP Server (Database): Executes query
Returns: {name: "Alice", email: "alice@example.com", ...}

Step 5: Response flows back
MCP Client: Receives response
Claude: Integrates result into context

Step 6: Claude continues reasoning
Claude: "The user is Alice. I can now help them with..."
```

---

## 1.4 MCP vs Other Approaches

### Approach 1: Direct Integration (No MCP)

```python
# Each tool is tightly integrated
from my_db import query_database
from my_api import call_api
from my_email import send_email

tools = [
    {
        "name": "query_database",
        "handler": query_database  # Direct Python function
    },
    {
        "name": "call_api",
        "handler": call_api
    },
    # ... more tools
]

# Pros:
# ✓ Simple for small projects
# ✓ No overhead
# Cons:
# ✗ Tightly coupled
# ✗ Can't reuse in other projects
# ✗ Hard to scale to many tools
# ✗ Must be same language as LLM client
```

### Approach 2: REST API (Generic)

```
Each tool becomes a REST endpoint
Claude → HTTP request → API endpoint → Tool
         ← HTTP response ←

Pros:
✓ Language-agnostic
✓ Can be remote

Cons:
✗ Manual request/response formatting
✗ No standardization (each endpoint is different)
✗ No tool discovery mechanism
✗ Harder to reason about contracts
```

### Approach 3: MCP (Standardized Protocol)

```
Each tool exposes MCP server
Claude → MCP request → MCP Server → Tool
         ← MCP response ←

Pros:
✓ Standardized (same pattern for all tools)
✓ Language-agnostic
✓ Tool discovery built-in
✓ Reusable (other LLMs can use same MCP servers)
✓ Composable (can chain MCP servers)
✓ Clear contracts (schema-based)

Cons:
✗ Slight additional complexity (worth it at scale)
✗ Ecosystem still growing (but rapidly)
```

### Comparison Table

| Aspect | Direct Integration | REST API | MCP |
|--------|---|---|---|
| Setup complexity | Low | Medium | Medium |
| Language support | Single language | Any | Any |
| Tool reusability | No | Partial | Yes |
| Standardization | None | Per-API | Full |
| Tool discovery | Manual | Manual | Automatic |
| Scaling (10+ tools) | ❌ Messy | ⚠️ Manageable | ✅ Clean |
| Ecosystem | None | Any REST | Growing MCP ecosystem |
| Security | Within process | Network security | MCP security model |

---

# Part 2: Building MCP Servers

## 2.1 MCP Server Basics

### What Is an MCP Server?

An MCP server is a process that:
1. Listens for MCP requests (over stdio or network)
2. Implements tools, resources, or prompts
3. Responds with results (following MCP protocol)

### Minimal MCP Server Example

```python
import json
import sys

class MCPServer:
    def __init__(self):
        self.tools = {}
    
    def register_tool(self, name: str, fn, schema: dict):
        """Register a tool with the server."""
        self.tools[name] = {
            "function": fn,
            "schema": schema
        }
    
    def handle_request(self, request: dict) -> dict:
        """Process an incoming MCP request."""
        method = request.get("method")
        
        if method == "initialize":
            return self._handle_initialize()
        
        elif method == "tools/list":
            return self._handle_list_tools()
        
        elif method == "tools/call":
            return self._handle_call_tool(request)
        
        else:
            return {"error": f"Unknown method: {method}"}
    
    def _handle_initialize(self) -> dict:
        """Respond to initialization."""
        return {
            "protocolVersion": "2024-11-05",
            "capabilities": {
                "tools": {}
            },
            "serverInfo": {
                "name": "my-mcp-server",
                "version": "1.0.0"
            }
        }
    
    def _handle_list_tools(self) -> dict:
        """List all available tools."""
        tools_list = []
        for name, tool_info in self.tools.items():
            tools_list.append({
                "name": name,
                "description": tool_info.get("description", ""),
                "inputSchema": tool_info["schema"]
            })
        return {"tools": tools_list}
    
    def _handle_call_tool(self, request: dict) -> dict:
        """Execute a tool."""
        tool_name = request["params"]["name"]
        arguments = request["params"]["arguments"]
        
        if tool_name not in self.tools:
            return {"error": f"Tool not found: {tool_name}"}
        
        try:
            result = self.tools[tool_name]["function"](**arguments)
            return {"content": [{"type": "text", "text": str(result)}]}
        except Exception as e:
            return {"error": str(e)}
    
    def run(self):
        """Main server loop."""
        while True:
            # Read request from stdin (MCP protocol)
            line = sys.stdin.readline()
            if not line:
                break
            
            request = json.loads(line)
            response = self.handle_request(request)
            
            # Send response to stdout
            print(json.dumps(response))
            sys.stdout.flush()

# Usage
server = MCPServer()

# Register a tool
def get_weather(location: str) -> str:
    return f"Weather in {location}: Sunny, 72°F"

server.register_tool(
    "get_weather",
    get_weather,
    {
        "type": "object",
        "properties": {
            "location": {"type": "string"}
        },
        "required": ["location"]
    }
)

# Run the server
server.run()
```

### How It Works

```
MCP Server starts listening
    ↓
Client connects (over stdio or TCP)
    ↓
Client sends: {"method": "initialize"}
Server responds: Server capabilities
    ↓
Client sends: {"method": "tools/list"}
Server responds: List of available tools
    ↓
Client sends: {"method": "tools/call", "params": {"name": "get_weather", "arguments": {"location": "Paris"}}}
Server responds: {"content": [...]}
    ↓
Repeat as needed
```

---

## 2.2 Implementing Tools via MCP

### Tool Schema Design

```python
# Tool schema defines the contract
weather_tool_schema = {
    "type": "object",
    "properties": {
        "location": {
            "type": "string",
            "description": "City name (e.g., 'Paris', 'Tokyo')"
        },
        "units": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"],
            "description": "Temperature units",
            "default": "fahrenheit"
        }
    },
    "required": ["location"]
}

# Schema validation
def validate_arguments(arguments: dict, schema: dict) -> tuple[bool, str]:
    """Validate tool arguments against schema."""
    
    # Check required fields
    for required_field in schema.get("required", []):
        if required_field not in arguments:
            return False, f"Missing required field: {required_field}"
    
    # Check types
    for field_name, field_value in arguments.items():
        if field_name not in schema.get("properties", {}):
            continue
        
        field_schema = schema["properties"][field_name]
        expected_type = field_schema.get("type")
        
        if expected_type == "string" and not isinstance(field_value, str):
            return False, f"{field_name} must be string"
        
        # Check enum constraints
        if "enum" in field_schema and field_value not in field_schema["enum"]:
            return False, f"{field_name} must be one of {field_schema['enum']}"
    
    return True, ""
```

### Tool Organization

```python
class ToolRegistry:
    """Manage multiple tools with schemas."""
    
    def __init__(self):
        self.tools = {}
    
    def register(self, name: str, description: str, schema: dict, handler):
        """Register a tool."""
        self.tools[name] = {
            "description": description,
            "schema": schema,
            "handler": handler
        }
    
    def get_schema(self, tool_name: str) -> dict:
        """Get tool schema for listing."""
        if tool_name not in self.tools:
            return None
        
        tool = self.tools[tool_name]
        return {
            "name": tool_name,
            "description": tool["description"],
            "inputSchema": tool["schema"]
        }
    
    def call(self, tool_name: str, arguments: dict) -> str:
        """Call a tool."""
        if tool_name not in self.tools:
            raise ValueError(f"Unknown tool: {tool_name}")
        
        tool = self.tools[tool_name]
        
        # Validate
        is_valid, error = self._validate(arguments, tool["schema"])
        if not is_valid:
            return f"Validation error: {error}"
        
        # Execute
        try:
            return tool["handler"](**arguments)
        except Exception as e:
            return f"Execution error: {e}"
    
    @staticmethod
    def _validate(arguments: dict, schema: dict) -> tuple[bool, str]:
        """Validate arguments against schema."""
        # Implementation as above
        pass

# Usage
registry = ToolRegistry()

registry.register(
    "get_weather",
    "Get current weather for a location",
    {
        "type": "object",
        "properties": {
            "location": {"type": "string"},
            "units": {"type": "string", "enum": ["celsius", "fahrenheit"]}
        },
        "required": ["location"]
    },
    lambda location, units="fahrenheit": f"Weather in {location}: Sunny, 72°{units[0].upper()}"
)

registry.register(
    "get_time",
    "Get current time",
    {"type": "object", "properties": {}},
    lambda: str(datetime.now())
)

# Now the server can use this registry
result = registry.call("get_weather", {"location": "Paris"})
print(result)  # "Weather in Paris: Sunny, 72°F"
```

---

## 2.3 Resource Management

### What Are Resources?

Resources are data exposed by an MCP server (not just callable tools).

```
Tools: Do something (action-oriented)
"get_weather" → Returns weather data

Resources: Access something (data-oriented)
"database://users" → Returns user data as resource
"file://documents/report.txt" → Returns file content
"api://github/repos" → Returns repository list
```

### Implementing Resources

```python
class ResourceServer:
    """MCP server that exposes resources."""
    
    def __init__(self):
        self.resources = {}
    
    def register_resource(self, uri: str, description: str, content_type: str, getter):
        """Register a resource URI."""
        self.resources[uri] = {
            "description": description,
            "contentType": content_type,
            "getter": getter
        }
    
    def handle_request(self, request: dict) -> dict:
        """Handle MCP requests."""
        method = request.get("method")
        
        if method == "resources/list":
            return self._list_resources()
        elif method == "resources/read":
            return self._read_resource(request)
        # ... other methods
    
    def _list_resources(self) -> dict:
        """List all available resources."""
        resources = []
        for uri, info in self.resources.items():
            resources.append({
                "uri": uri,
                "name": uri.split("://")[-1],  # Extract name from URI
                "description": info["description"],
                "mimeType": info["contentType"]
            })
        return {"resources": resources}
    
    def _read_resource(self, request: dict) -> dict:
        """Read a specific resource."""
        uri = request["params"]["uri"]
        
        if uri not in self.resources:
            return {"error": f"Resource not found: {uri}"}
        
        try:
            content = self.resources[uri]["getter"]()
            return {
                "contents": [{
                    "uri": uri,
                    "mimeType": self.resources[uri]["contentType"],
                    "text": str(content)
                }]
            }
        except Exception as e:
            return {"error": str(e)}

# Usage
server = ResourceServer()

# Expose database data as resource
def get_users_data():
    return json.dumps([
        {"id": 1, "name": "Alice", "email": "alice@example.com"},
        {"id": 2, "name": "Bob", "email": "bob@example.com"}
    ])

server.register_resource(
    "database://users",
    "User database records",
    "application/json",
    get_users_data
)

# Expose file data as resource
def get_file_content():
    with open("documents/report.txt") as f:
        return f.read()

server.register_resource(
    "file://documents/report.txt",
    "Latest quarterly report",
    "text/plain",
    get_file_content
)
```

---

## 2.4 Prompts & Templates

### What Are Prompt Resources?

Prompt resources are pre-built prompts that an MCP server can suggest to LLMs.

```
Example: Database MCP server suggests prompts like:
- "How to write efficient SQL queries"
- "Best practices for database design"
- "Query optimization techniques"

An LLM can ask the server for these prompts, then use them
to improve its responses or guide its reasoning.
```

### Implementing Prompts

```python
class PromptServer:
    """MCP server that exposes prompt templates."""
    
    def __init__(self):
        self.prompts = {}
    
    def register_prompt(self, name: str, description: str, template: str, arguments: dict = None):
        """Register a prompt template."""
        self.prompts[name] = {
            "description": description,
            "template": template,
            "argumentSchema": arguments or {"type": "object", "properties": {}}
        }
    
    def handle_request(self, request: dict) -> dict:
        method = request.get("method")
        
        if method == "prompts/list":
            return self._list_prompts()
        elif method == "prompts/get":
            return self._get_prompt(request)
    
    def _list_prompts(self) -> dict:
        """List available prompts."""
        prompts = []
        for name, info in self.prompts.items():
            prompts.append({
                "name": name,
                "description": info["description"],
                "argumentSchema": info["argumentSchema"]
            })
        return {"prompts": prompts}
    
    def _get_prompt(self, request: dict) -> dict:
        """Get a specific prompt, with arguments filled in."""
        name = request["params"]["name"]
        arguments = request["params"].get("arguments", {})
        
        if name not in self.prompts:
            return {"error": f"Prompt not found: {name}"}
        
        template = self.prompts[name]["template"]
        
        # Fill in template with arguments
        prompt_text = template.format(**arguments)
        
        return {
            "messages": [{
                "role": "user",
                "content": prompt_text
            }]
        }

# Usage
server = PromptServer()

# Register a prompt for database queries
server.register_prompt(
    "write_efficient_query",
    "Template for writing efficient SQL queries",
    """You are an SQL optimization expert.

Task: Write an efficient SQL query for the following requirement:
{requirement}

Guidelines:
- Use appropriate indexes
- Avoid N+1 queries
- Optimize JOIN conditions
- Consider query execution plan

Please provide the SQL query and explain the optimization choices.""",
    {
        "type": "object",
        "properties": {
            "requirement": {"type": "string", "description": "What data do you need?"}
        },
        "required": ["requirement"]
    }
)

# When LLM asks for this prompt:
# GET prompts/write_efficient_query with arguments {"requirement": "Get all users and their orders"}
# Server returns the filled-in prompt
```

---

# Part 3: MCP Clients & Integration

## 3.1 MCP Client Patterns

### Basic Client Implementation

```python
import json
import subprocess
from typing import Any, dict

class MCPClient:
    """Generic MCP client for communicating with MCP servers."""
    
    def __init__(self, server_command: str):
        """
        Initialize MCP client.
        server_command: Command to start the MCP server (e.g., "python my_server.py")
        """
        self.process = subprocess.Popen(
            server_command,
            shell=True,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            text=True,
            bufsize=1
        )
        self.next_id = 1
    
    def send_request(self, method: str, params: dict = None) -> dict:
        """Send an MCP request and receive response."""
        
        request = {
            "jsonrpc": "2.0",
            "id": self.next_id,
            "method": method,
            "params": params or {}
        }
        self.next_id += 1
        
        # Send request
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()
        
        # Receive response
        response_line = self.process.stdout.readline()
        response = json.loads(response_line)
        
        if "error" in response:
            raise Exception(f"MCP Error: {response['error']}")
        
        return response.get("result", response)
    
    def initialize(self) -> dict:
        """Initialize the MCP server."""
        return self.send_request("initialize")
    
    def list_tools(self) -> list:
        """Get list of available tools."""
        response = self.send_request("tools/list")
        return response.get("tools", [])
    
    def call_tool(self, tool_name: str, arguments: dict) -> str:
        """Call a tool on the MCP server."""
        response = self.send_request("tools/call", {
            "name": tool_name,
            "arguments": arguments
        })
        
        # Extract text from content array
        if "content" in response:
            return response["content"][0].get("text", "")
        return str(response)
    
    def list_resources(self) -> list:
        """Get list of available resources."""
        response = self.send_request("resources/list")
        return response.get("resources", [])
    
    def read_resource(self, uri: str) -> str:
        """Read a specific resource."""
        response = self.send_request("resources/read", {"uri": uri})
        
        if "contents" in response:
            return response["contents"][0].get("text", "")
        return str(response)
    
    def shutdown(self):
        """Shutdown the MCP server."""
        self.process.terminate()
        self.process.wait()

# Usage
client = MCPClient("python my_mcp_server.py")

# Initialize
client.initialize()

# List and call tools
tools = client.list_tools()
print(f"Available tools: {[t['name'] for t in tools]}")

result = client.call_tool("get_weather", {"location": "Paris"})
print(f"Result: {result}")

client.shutdown()
```

### Multi-Server Client

```python
class MultiServerMCPClient:
    """Client that manages multiple MCP servers."""
    
    def __init__(self):
        self.servers = {}  # server_name -> MCPClient
        self.tool_to_server = {}  # tool_name -> server_name
    
    def connect_server(self, name: str, command: str):
        """Connect to an MCP server."""
        client = MCPClient(command)
        client.initialize()
        self.servers[name] = client
        
        # Index tools
        for tool in client.list_tools():
            self.tool_to_server[tool["name"]] = name
    
    def call_tool(self, tool_name: str, arguments: dict) -> str:
        """Call a tool (automatically routes to correct server)."""
        if tool_name not in self.tool_to_server:
            raise ValueError(f"Tool not found: {tool_name}")
        
        server_name = self.tool_to_server[tool_name]
        return self.servers[server_name].call_tool(tool_name, arguments)
    
    def get_all_tools(self) -> dict:
        """Get all tools from all servers."""
        all_tools = {}
        for server_name, client in self.servers.items():
            for tool in client.list_tools():
                all_tools[tool["name"]] = {
                    "server": server_name,
                    "schema": tool
                }
        return all_tools

# Usage
multi_client = MultiServerMCPClient()

# Connect multiple servers
multi_client.connect_server("database", "python database_mcp.py")
multi_client.connect_server("weather", "python weather_mcp.py")
multi_client.connect_server("files", "python files_mcp.py")

# Call tools (client figures out which server)
result = multi_client.call_tool("query_database", {"query": "SELECT * FROM users"})
result = multi_client.call_tool("get_weather", {"location": "Paris"})
result = multi_client.call_tool("read_file", {"path": "data.txt"})

# Get all available tools
all_tools = multi_client.get_all_tools()
```

---

## 3.2 Claude + MCP Integration

### Claude with MCP Tools

```python
from anthropic import Anthropic

class ClaudeWithMCP:
    """Claude using MCP servers as tools."""
    
    def __init__(self, mcp_client):
        self.client = Anthropic()
        self.mcp = mcp_client
        self.model = "claude-opus-4-8"
    
    def run_agent(self, user_query: str, max_iterations: int = 10) -> str:
        """Run Claude as an agent with MCP tools."""
        
        # Get MCP tools in Claude format
        claude_tools = self._convert_mcp_tools_to_claude()
        
        messages = [{"role": "user", "content": user_query}]
        
        for iteration in range(max_iterations):
            # Call Claude with tools
            response = self.client.messages.create(
                model=self.model,
                max_tokens=2048,
                tools=claude_tools,
                messages=messages
            )
            
            # Check if done
            if response.stop_reason == "end_turn":
                # Extract final answer
                for block in response.content:
                    if hasattr(block, "text"):
                        return block.text
                return ""
            
            # Extract tool calls
            tool_calls = [b for b in response.content if b.type == "tool_use"]
            
            if not tool_calls:
                # No tools called, return response
                for block in response.content:
                    if hasattr(block, "text"):
                        return block.text
                return ""
            
            # Add assistant response to history
            messages.append({"role": "assistant", "content": response.content})
            
            # Execute tools via MCP
            tool_results = []
            for tool_call in tool_calls:
                try:
                    result = self.mcp.call_tool(tool_call.name, tool_call.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": tool_call.id,
                        "content": result
                    })
                except Exception as e:
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": tool_call.id,
                        "content": f"Error: {e}",
                        "is_error": True
                    })
            
            # Add results back to messages
            messages.append({"role": "user", "content": tool_results})
        
        return "Max iterations reached"
    
    def _convert_mcp_tools_to_claude(self) -> list:
        """Convert MCP tools to Claude tool format."""
        tools = []
        for mcp_tool in self.mcp.list_tools():
            tools.append({
                "name": mcp_tool["name"],
                "description": mcp_tool["description"],
                "input_schema": mcp_tool["inputSchema"]
            })
        return tools

# Usage
mcp_client = MCPClient("python my_mcp_server.py")
claude_agent = ClaudeWithMCP(mcp_client)

result = claude_agent.run_agent("Get the weather in Paris and current time")
print(result)
```

---

## 3.3 Handling Errors & Timeouts

### Error Handling Patterns

```python
class RobustMCPClient:
    """MCP client with comprehensive error handling."""
    
    def __init__(self, server_command: str, timeout_sec: float = 10.0):
        self.server_command = server_command
        self.timeout = timeout_sec
        self.process = None
    
    def call_tool_with_retry(self, tool_name: str, arguments: dict, max_retries: int = 3) -> str:
        """Call tool with automatic retry and timeout handling."""
        
        for attempt in range(max_retries):
            try:
                # Start fresh connection for each attempt
                self._ensure_connected()
                
                # Call tool with timeout
                import signal
                
                def timeout_handler(signum, frame):
                    raise TimeoutError(f"Tool call exceeded {self.timeout}s timeout")
                
                signal.signal(signal.SIGALRM, timeout_handler)
                signal.alarm(int(self.timeout))
                
                try:
                    result = self._call_tool(tool_name, arguments)
                    signal.alarm(0)  # Cancel alarm
                    return result
                except TimeoutError as e:
                    signal.alarm(0)  # Cancel alarm
                    raise e
            
            except (TimeoutError, BrokenPipeError, IOError) as e:
                # Transient error, retry
                if attempt < max_retries - 1:
                    import time
                    wait_time = 2 ** attempt  # Exponential backoff
                    print(f"Attempt {attempt+1} failed: {e}. Retrying in {wait_time}s...")
                    time.sleep(wait_time)
                    self._reset_connection()
                else:
                    raise
            
            except ValueError as e:
                # Permanent error (invalid arguments), don't retry
                raise e
    
    def _ensure_connected(self):
        """Ensure server is running."""
        if self.process is None or self.process.poll() is not None:
            self.process = subprocess.Popen(
                self.server_command,
                shell=True,
                stdin=subprocess.PIPE,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True
            )
    
    def _reset_connection(self):
        """Reset connection to server."""
        if self.process:
            self.process.terminate()
            self.process = None
    
    def _call_tool(self, tool_name: str, arguments: dict) -> str:
        """Internal tool call (without retry logic)."""
        request = {
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/call",
            "params": {"name": tool_name, "arguments": arguments}
        }
        
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()
        
        response_line = self.process.stdout.readline()
        response = json.loads(response_line)
        
        if "error" in response:
            raise ValueError(f"Tool error: {response['error']}")
        
        return response["result"]["content"][0]["text"]
```

---

## 3.4 Security Considerations

### Input Validation

```python
def validate_tool_arguments(arguments: dict, schema: dict) -> tuple[bool, str]:
    """Validate tool arguments against schema."""
    
    # Prevent injection attacks
    if isinstance(arguments, str):
        # Reject string arguments (should be dict)
        return False, "Arguments must be a dictionary"
    
    # Check required fields
    for required in schema.get("required", []):
        if required not in arguments:
            return False, f"Missing required field: {required}"
    
    # Type validation
    for field_name, field_value in arguments.items():
        if field_name not in schema.get("properties", {}):
            return False, f"Unknown field: {field_name}"
        
        field_schema = schema["properties"][field_name]
        expected_type = field_schema.get("type")
        
        # Type checking
        type_map = {
            "string": str,
            "number": (int, float),
            "integer": int,
            "boolean": bool,
            "array": list,
            "object": dict
        }
        
        if expected_type in type_map and not isinstance(field_value, type_map[expected_type]):
            return False, f"Field {field_name} must be {expected_type}"
        
        # Enum validation
        if "enum" in field_schema and field_value not in field_schema["enum"]:
            return False, f"Field {field_name} must be one of {field_schema['enum']}"
        
        # String length limits (prevent abuse)
        if expected_type == "string" and len(str(field_value)) > 10000:
            return False, f"Field {field_name} exceeds max length"
    
    return True, ""
```

### Sandboxing

```python
import subprocess
import tempfile
import os

class SandboxedMCPServer:
    """Run MCP server in sandbox for security."""
    
    def __init__(self, server_script: str):
        self.server_script = server_script
        self.process = None
    
    def run_sandboxed(self):
        """Run server in a restricted environment."""
        
        # Create temporary directory for sandboxed execution
        with tempfile.TemporaryDirectory() as tmpdir:
            # Set up restricted environment
            env = os.environ.copy()
            env["HOME"] = tmpdir  # Restrict home directory
            env["TMPDIR"] = tmpdir
            
            # Run server with restrictions
            # (Using --timeout, --memory limits, chroot, etc. depends on OS)
            cmd = f"timeout 300 python {self.server_script}"  # 5 minute timeout
            
            self.process = subprocess.Popen(
                cmd,
                stdin=subprocess.PIPE,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                env=env,
                # Additional security: resource limits
                preexec_fn=self._set_resource_limits if hasattr(os, 'setrlimit') else None
            )
    
    @staticmethod
    def _set_resource_limits():
        """Set resource limits for server process."""
        import resource
        
        # Limit memory to 512MB
        resource.setrlimit(resource.RLIMIT_AS, (512 * 1024 * 1024, 512 * 1024 * 1024))
        
        # Limit CPU time to 60 seconds
        resource.setrlimit(resource.RLIMIT_CPU, (60, 60))
        
        # Limit file size to 100MB
        resource.setrlimit(resource.RLIMIT_FSIZE, (100 * 1024 * 1024, 100 * 1024 * 1024))
```

---

# Part 4: Multi-Agent Fundamentals

## 4.1 What Are Multi-Agent Systems?

### Definition

A **multi-agent system** is a collection of independent agents that:
1. Work toward shared or individual goals
2. Communicate with each other
3. Coordinate their actions
4. Can be specialized for different tasks

### Why Multiple Agents?

```
Single Agent:
✓ Simple
✗ Must know everything
✗ Can't specialize
✗ Slower for parallel tasks

Multi-Agent:
✓ Can specialize (expert agents)
✓ Can work in parallel
✓ More resilient (if one fails, others continue)
✗ Coordination complexity

Example: Analyzing a research paper

Single Agent:
"Summarize, extract key findings, check citations"
→ Agent tries to do everything
→ May miss details
→ Takes a long time

Multi-Agent:
Agent 1 (Summarizer): Summarizes paper
Agent 2 (Researcher): Looks up related work
Agent 3 (Critic): Checks methodology
Agent 4 (Synthesizer): Combines insights
→ Each agent is specialized
→ Can work in parallel
→ Better overall analysis
```

---

## 4.2 Agent Types & Roles

### Common Agent Roles

```
1. Manager Agent
   Role: Decompose tasks, coordinate, synthesize
   Example: "Break this down into subtasks"

2. Specialist Agents
   Role: Deep expertise in one domain
   Example: "I'm a Python code review expert"

3. Researcher Agent
   Role: Find and verify information
   Example: "I search for relevant documents"

4. Critic Agent
   Role: Check for errors and inconsistencies
   Example: "I verify claims and find logical flaws"

5. Synthesizer Agent
   Role: Combine multiple perspectives
   Example: "I merge insights from all sources"

6. Executor Agent
   Role: Take action in external systems
   Example: "I run code, make API calls, update databases"
```

### Agent Specialization

```python
class SpecializedAgent:
    """An agent with specific expertise."""
    
    def __init__(self, 
                 name: str,
                 role: str,
                 expertise_areas: list[str],
                 system_prompt: str):
        self.name = name
        self.role = role
        self.expertise_areas = expertise_areas
        self.system_prompt = system_prompt
    
    def can_handle(self, task: str) -> bool:
        """Can this agent handle this task?"""
        task_lower = task.lower()
        return any(
            area.lower() in task_lower
            for area in self.expertise_areas
        )
    
    def process(self, input_text: str) -> str:
        """Process input using specialized knowledge."""
        response = self.client.messages.create(
            model="claude-opus-4-8",
            max_tokens=2048,
            system=self.system_prompt,
            messages=[{"role": "user", "content": input_text}]
        )
        return response.content[0].text

# Create specialized agents
code_reviewer = SpecializedAgent(
    name="Code Reviewer",
    role="Security and best practices reviewer",
    expertise_areas=["code", "security", "performance", "python", "javascript"],
    system_prompt="""You are an expert code reviewer specializing in:
    - Security vulnerabilities
    - Performance optimization
    - Best practices
    - Readability and maintainability
    
    Provide actionable feedback with specific examples."""
)

researcher = SpecializedAgent(
    name="Researcher",
    role="Information gathering and analysis",
    expertise_areas=["research", "evidence", "data", "analysis"],
    system_prompt="""You are a research expert who:
    - Finds reliable sources
    - Verifies facts
    - Identifies patterns in data
    - Synthesizes findings
    
    Always cite sources and distinguish fact from interpretation."""
)
```

---

## 4.3 Communication Patterns

### Direct Communication

```
Agent A → Message → Agent B
"Please review this code"
←  Response ←

Simple but Agent B must be ready to respond
```

### Message Queue

```
Agent A → Queue → Worker processes → Agent B
Message is stored until Agent B is ready
Better for asynchronous communication
```

### Broadcast

```
Agent A broadcasts → All listening agents receive
Agent B: "I'll handle this"
Agent C: "I'll help with that part"

Good for: Tasks that multiple agents can contribute to
```

### Implementation

```python
class MessageQueue:
    """Message queue for agent communication."""
    
    def __init__(self):
        self.messages = []
        self.subscribers = {}  # agent_name -> callback
    
    def subscribe(self, agent_name: str, callback):
        """Agent subscribes to messages."""
        self.subscribers[agent_name] = callback
    
    def send_message(self, from_agent: str, to_agent: str, content: str):
        """Send message from one agent to another."""
        message = {
            "from": from_agent,
            "to": to_agent,
            "content": content,
            "timestamp": datetime.now()
        }
        
        # If recipient is online, deliver immediately
        if to_agent in self.subscribers:
            self.subscribers[to_agent](message)
        else:
            # Otherwise queue it
            self.messages.append(message)
    
    def broadcast(self, from_agent: str, content: str, topic: str = "general"):
        """Broadcast message to all agents interested in topic."""
        message = {
            "from": from_agent,
            "to": "all",
            "content": content,
            "topic": topic,
            "timestamp": datetime.now()
        }
        
        # Send to all subscribers
        for agent_name, callback in self.subscribers.items():
            if agent_name != from_agent:  # Don't send to self
                callback(message)
    
    def get_messages(self, agent_name: str) -> list:
        """Get queued messages for an agent."""
        messages = [m for m in self.messages if m["to"] == agent_name]
        # Remove delivered messages from queue
        self.messages = [m for m in self.messages if m["to"] != agent_name]
        return messages
```

---

## 4.4 Coordination Mechanisms

### Task Distribution

```python
class TaskDistributor:
    """Distribute tasks to agents based on capability."""
    
    def __init__(self, agents: list):
        self.agents = agents
    
    def dispatch_task(self, task: str) -> dict:
        """Find best agent(s) for task and dispatch."""
        
        # Find agents that can handle this task
        capable_agents = [
            agent for agent in self.agents
            if agent.can_handle(task)
        ]
        
        if not capable_agents:
            return {"error": "No agent can handle this task"}
        
        # Choose best agent (simplest: first capable)
        chosen_agent = capable_agents[0]
        
        # Execute task
        result = chosen_agent.process(task)
        
        return {
            "agent": chosen_agent.name,
            "task": task,
            "result": result
        }
```

### Consensus Mechanism

```python
class ConsensusManager:
    """Reach consensus among multiple agents."""
    
    def __init__(self, agents: list):
        self.agents = agents
    
    def get_consensus(self, question: str) -> dict:
        """Get all agents' perspectives and find consensus."""
        
        # Ask all agents
        responses = {}
        for agent in self.agents:
            response = agent.process(question)
            responses[agent.name] = response
        
        # Synthesize using an additional synthesizer agent
        synthesis_prompt = f"""
        Multiple agents have provided their perspectives:
        
        {chr(10).join([f"{name}: {resp}" for name, resp in responses.items()])}
        
        Synthesize these perspectives into a coherent conclusion.
        Note areas of agreement and disagreement.
        """
        
        synthesis = self.synthesizer.process(synthesis_prompt)
        
        return {
            "individual_responses": responses,
            "consensus": synthesis
        }
```

---

# Part 5: Multi-Agent Architectures

## 5.1 Manager-Worker Pattern

### Architecture

```
┌─────────────┐
│   Manager   │  Decomposes task
│   Agent     │  Coordinates workers
│             │  Synthesizes results
└──────┬──────┘
       │
       ├──────────────────┬──────────────────┬──────────────────┐
       │                  │                  │                  │
       ▼                  ▼                  ▼                  ▼
    ┌──────┐          ┌──────┐          ┌──────┐          ┌──────┐
    │Worker│          │Worker│          │Worker│          │Worker│
    │  1   │          │  2   │          │  3   │          │  4   │
    │Code  │          │Test  │          │Docs  │          │Design│
    │Expert│          │Expert│          │Expert│          │Expert│
    └──┬───┘          └──┬───┘          └──┬───┘          └──┬───┘
       │                  │                  │                  │
       │                  ▼                  ▼                  ▼
       └──────────────────────────────────────────────────────┘
                    Results back to Manager
```

### Implementation

```python
class ManagerWorkerSystem:
    """Manager-worker architecture for task decomposition."""
    
    def __init__(self, workers: dict[str, SpecializedAgent]):
        self.workers = workers  # {worker_name: agent}
        self.manager = self._create_manager()
    
    def _create_manager(self) -> SpecializedAgent:
        """Create manager agent that coordinates workers."""
        return SpecializedAgent(
            name="Manager",
            role="Task coordinator",
            expertise_areas=["coordination", "planning"],
            system_prompt="""You are a task manager. Your job is to:
            1. Decompose complex tasks into subtasks
            2. Assign each subtask to the best worker
            3. Coordinate worker execution
            4. Synthesize results
            
            Workers available: """ + ", ".join(self.workers.keys())
        )
    
    def execute(self, main_task: str) -> str:
        """Execute a task using manager-worker pattern."""
        
        # Step 1: Manager decomposes task
        decomposition_prompt = f"""
        Task: {main_task}
        
        Available workers: {list(self.workers.keys())}
        
        Decompose this task into subtasks. For each subtask, specify:
        1. Subtask description
        2. Which worker should handle it
        3. Why this worker is best
        """
        
        decomposition = self.manager.process(decomposition_prompt)
        print(f"Task decomposed:\n{decomposition}")
        
        # Step 2: Parse decomposition and execute subtasks
        # (In real code, parse more carefully)
        subtasks = self._parse_subtasks(decomposition)
        
        results = {}
        for subtask in subtasks:
            worker_name = self._assign_worker(subtask)
            result = self.workers[worker_name].process(subtask)
            results[worker_name] = result
        
        # Step 3: Manager synthesizes results
        synthesis_prompt = f"""
        Original task: {main_task}
        
        Subtask results:
        {chr(10).join([f"{name}: {res}" for name, res in results.items()])}
        
        Synthesize these results into a final answer.
        """
        
        final_result = self.manager.process(synthesis_prompt)
        return final_result
    
    def _parse_subtasks(self, decomposition: str) -> list[str]:
        """Parse decomposition to extract subtasks."""
        # Simplified parsing
        return decomposition.split("\n")
    
    def _assign_worker(self, subtask: str) -> str:
        """Find best worker for subtask."""
        for worker_name, worker in self.workers.items():
            if worker.can_handle(subtask):
                return worker_name
        return list(self.workers.keys())[0]  # Default

# Usage
workers = {
    "code_expert": code_reviewer,
    "researcher": researcher,
    "critic": critic_agent
}

system = ManagerWorkerSystem(workers)
result = system.execute("Review and improve this research proposal")
print(result)
```

---

## 5.2 Hierarchical Agents

### Multi-Level Hierarchy

```
                    ┌──────────────┐
                    │   Overseer   │ (Level 0: Big picture)
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
    ┌────────┐        ┌────────┐        ┌────────┐
    │Manager │        │Manager │        │Manager │
    │ (Engg) │        │(Design)│        │(QA)    │
    └───┬────┘        └───┬────┘        └───┬────┘
        │                  │                  │
    ┌───┴───┐          ┌───┴───┐        ┌───┴───┐
    │       │          │       │        │       │
    ▼       ▼          ▼       ▼        ▼       ▼
   Dev1    Dev2      Des1    Des2      QA1    QA2
```

### Implementation

```python
class HierarchicalAgent:
    """Agent in a hierarchy with parent and children."""
    
    def __init__(self, name: str, level: int, role: str):
        self.name = name
        self.level = level
        self.role = role
        self.parent = None
        self.children = []
    
    def add_child(self, child: 'HierarchicalAgent'):
        """Add subordinate agent."""
        child.parent = self
        self.children.append(child)
    
    def delegate(self, task: str) -> dict:
        """Delegate task to appropriate child."""
        
        # Ask children who can handle it
        capable = [
            child for child in self.children
            if child.can_handle(task)
        ]
        
        if not capable:
            # Must handle it myself
            return {"agent": self.name, "result": self._execute(task)}
        
        # Delegate to most capable
        delegated_to = capable[0]
        return {
            "delegated_from": self.name,
            "delegated_to": delegated_to.name,
            "result": delegated_to.delegate(task)
        }
    
    def _execute(self, task: str) -> str:
        """Execute task at this level."""
        # Implementation specific to this agent's expertise
        pass
    
    def report_up(self, message: str):
        """Report status to parent."""
        if self.parent:
            self.parent.receive_report(self.name, message)
    
    def receive_report(self, from_child: str, message: str):
        """Receive report from child agent."""
        print(f"{self.name} received report from {from_child}: {message}")
        
        # Can decide to report further up
        if self.parent:
            self.report_up(f"From {from_child}: {message}")
```

---

## 5.3 Peer-to-Peer Collaboration

### Collaborative Problem Solving

```
Agent A ←→ Agent B
   ↕        ↕
Agent C ←→ Agent D

All agents work together, equal status
No central coordinator
```

### Implementation

```python
class CollaborativeAgent:
    """Agent that works collaboratively with peers."""
    
    def __init__(self, name: str, perspective: str):
        self.name = name
        self.perspective = perspective  # Unique viewpoint
        self.peers = []
        self.shared_context = {}
    
    def add_peer(self, peer: 'CollaborativeAgent'):
        """Add another agent to collaborate with."""
        self.peers.append(peer)
    
    def collaborate(self, problem: str) -> str:
        """Collaborate with peers to solve problem."""
        
        # Step 1: Each agent provides their perspective
        perspectives = {}
        perspectives[self.name] = self._analyze(problem)
        
        for peer in self.peers:
            perspectives[peer.name] = peer._analyze(problem)
        
        # Step 2: Discuss and refine
        refined = self._synthesize(perspectives)
        
        # Step 3: Feed back to peers for further refinement
        for iteration in range(3):  # 3 rounds of refinement
            peer_feedback = []
            for peer in self.peers:
                feedback = peer._critique(refined)
                peer_feedback.append(feedback)
            
            refined = self._incorporate_feedback(refined, peer_feedback)
        
        return refined
    
    def _analyze(self, problem: str) -> str:
        """Analyze problem from this agent's perspective."""
        prompt = f"""
        From your perspective as a {self.perspective} expert:
        {problem}
        
        Provide your unique insight and analysis.
        """
        # Call LLM
        return "analysis from perspective"
    
    def _critique(self, proposal: str) -> str:
        """Critique a proposal from this agent's viewpoint."""
        prompt = f"""
        From your perspective as a {self.perspective} expert,
        critique this proposal:
        {proposal}
        
        What are the strengths and weaknesses?
        """
        # Call LLM
        return "critique"
    
    def _synthesize(self, perspectives: dict) -> str:
        """Integrate all perspectives."""
        prompt = f"""
        Synthesize these perspectives into a unified recommendation:
        {perspectives}
        
        Highlight agreements, conflicts, and how to reconcile them.
        """
        # Call LLM
        return "synthesis"
    
    def _incorporate_feedback(self, proposal: str, feedback: list) -> str:
        """Refine proposal based on feedback."""
        prompt = f"""
        Original proposal: {proposal}
        
        Feedback:
        {feedback}
        
        Refine the proposal to address the feedback.
        """
        # Call LLM
        return "refined proposal"

# Usage
analyst = CollaborativeAgent("Analyst", "quantitative analysis")
strategist = CollaborativeAgent("Strategist", "business strategy")
critic = CollaborativeAgent("Critic", "critical thinking")

analyst.add_peer(strategist)
analyst.add_peer(critic)
strategist.add_peer(analyst)
strategist.add_peer(critic)
critic.add_peer(analyst)
critic.add_peer(strategist)

result = analyst.collaborate("Should we enter the Japanese market?")
print(result)
```

---

## 5.4 Specialized Agent Teams

### Team Composition

```
Project Team:
- Project Manager: Coordinates timeline and resources
- Technical Lead: Makes architecture decisions
- Developer: Implements features
- QA Lead: Plans testing
- Designer: Ensures UX quality

Each agent specializes in one area
But collaborates with others
```

### Implementation

```python
class AgentTeam:
    """A team of specialized agents working on a project."""
    
    def __init__(self, project_name: str):
        self.project_name = project_name
        self.members = {}  # role -> agent
        self.shared_context = {
            "project_name": project_name,
            "timeline": None,
            "budget": None,
            "constraints": []
        }
    
    def add_member(self, role: str, agent: SpecializedAgent):
        """Add team member."""
        self.members[role] = agent
    
    def kickoff_meeting(self, project_brief: str):
        """Start project with all team members aligned."""
        
        # Manager reads brief and sets direction
        manager = self.members.get("project_manager")
        if manager:
            direction = manager.process(f"Project brief: {project_brief}")
            self.shared_context["direction"] = direction
        
        # Each member contributes initial assessment
        assessments = {}
        for role, agent in self.members.items():
            if role != "project_manager":
                assessment = agent.process(f"""
                Project brief: {project_brief}
                
                Manager's direction: {self.shared_context.get('direction', '')}
                
                From your perspective as {role}, what are the key considerations?
                """)
                assessments[role] = assessment
                self.shared_context[f"{role}_assessment"] = assessment
        
        return assessments
    
    def execute_phase(self, phase: str) -> dict:
        """Execute a project phase with team collaboration."""
        
        results = {}
        
        for role, agent in self.members.items():
            prompt = f"""
            Project: {self.project_name}
            Phase: {phase}
            
            Team assessments:
            {chr(10).join([f"{r}: {a}" for r, a in self.shared_context.items()])}
            
            As the {role}, what are your deliverables for this phase?
            """
            
            result = agent.process(prompt)
            results[role] = result
            self.shared_context[f"{role}_deliverable_{phase}"] = result
        
        return results
    
    def retrospective(self) -> str:
        """Team reviews what went well and what to improve."""
        
        prompt = f"""
        Project: {self.project_name}
        
        Deliverables by role:
        {chr(10).join([k: v for k, v in self.shared_context.items() if 'deliverable' in k])}
        
        As a team, what worked well? What could improve?
        """
        
        # Have project manager (or rotate) lead retrospective
        facilitator = self.members.get("project_manager")
        if not facilitator:
            facilitator = list(self.members.values())[0]
        
        return facilitator.process(prompt)
```

---

# Part 6: Production Patterns

## 6.1 Scalability & Performance

### Parallel Execution

```python
import asyncio

class ParallelAgentExecutor:
    """Execute multiple agents in parallel."""
    
    async def execute_agents_parallel(self, agents: list, task: str) -> dict:
        """Run multiple agents on same task in parallel."""
        
        # Create async tasks
        tasks = [
            self._run_agent_async(agent, task)
            for agent in agents
        ]
        
        # Wait for all to complete
        results = await asyncio.gather(*tasks)
        
        return {
            agent.name: result
            for agent, result in zip(agents, results)
        }
    
    async def _run_agent_async(self, agent: SpecializedAgent, task: str) -> str:
        """Run a single agent asynchronously."""
        # Would integrate with async LLM API
        return agent.process(task)

# Usage
executor = ParallelAgentExecutor()

agents = [analyst, researcher, critic]
results = asyncio.run(executor.execute_agents_parallel(agents, task))

# Instead of serial: time = 3 * 30s = 90s
# With parallel: time = max(30s, 30s, 30s) = 30s
```

### Load Balancing

```python
class AgentPool:
    """Pool of agent instances for load balancing."""
    
    def __init__(self, agent_type: type, pool_size: int = 5):
        self.agents = [agent_type() for _ in range(pool_size)]
        self.queue = []  # Tasks waiting for agents
        self.in_progress = {}  # task_id -> agent
    
    def submit_task(self, task_id: str, task: str) -> str:
        """Submit task to least-busy agent."""
        
        # Find idle agent
        busy_agents = set(self.in_progress.values())
        idle = [a for a in self.agents if a not in busy_agents]
        
        if idle:
            agent = idle[0]
        else:
            # Queue the task
            self.queue.append((task_id, task))
            return f"Task {task_id} queued"
        
        # Execute
        self.in_progress[task_id] = agent
        result = agent.process(task)
        del self.in_progress[task_id]
        
        # Process queued task if any
        if self.queue:
            next_id, next_task = self.queue.pop(0)
            self.submit_task(next_id, next_task)
        
        return result
```

---

## 6.2 Reliability & Fault Tolerance

### Agent Health Monitoring

```python
class AgentHealthMonitor:
    """Monitor agent health and performance."""
    
    def __init__(self):
        self.health_scores = {}  # agent_name -> score
        self.error_counts = {}
        self.response_times = {}
    
    def record_execution(self, agent_name: str, success: bool, response_time_ms: float):
        """Record execution metrics."""
        
        # Track response time
        if agent_name not in self.response_times:
            self.response_times[agent_name] = []
        self.response_times[agent_name].append(response_time_ms)
        
        # Track errors
        if not success:
            self.error_counts[agent_name] = self.error_counts.get(agent_name, 0) + 1
        
        # Calculate health score
        self._update_health_score(agent_name)
    
    def _update_health_score(self, agent_name: str):
        """Calculate agent health (0-1)."""
        
        error_rate = self.error_counts.get(agent_name, 0) / max(1, len(self.response_times.get(agent_name, [])))
        avg_latency = sum(self.response_times.get(agent_name, [0])) / max(1, len(self.response_times.get(agent_name, [])))
        
        # Health = (1 - error_rate) * (1 - min(latency/1000, 1))
        health = (1 - error_rate) * max(0, 1 - (avg_latency / 5000))
        
        self.health_scores[agent_name] = health
        
        if health < 0.5:
            print(f"⚠️  {agent_name} health degraded: {health:.2f}")
    
    def get_best_agent(self, available_agents: list) -> str:
        """Get healthiest available agent."""
        return max(
            available_agents,
            key=lambda a: self.health_scores.get(a.name, 0.5)
        )

monitor = AgentHealthMonitor()

# After each agent execution
monitor.record_execution("agent_a", success=True, response_time_ms=245)
monitor.record_execution("agent_b", success=False, response_time_ms=2340)  # Error

# Choose agent for next task
best = monitor.get_best_agent([agent_a, agent_b])
```

---

## 6.3 Monitoring & Observability

### Agent Tracing

```python
import json
from datetime import datetime

class AgentTracer:
    """Trace all agent interactions for observability."""
    
    def __init__(self, log_file: str = "agent_trace.jsonl"):
        self.log_file = log_file
        self.traces = []
    
    def trace_execution(self, agent_name: str, task: str, result: str, duration_ms: float):
        """Log agent execution."""
        
        trace = {
            "timestamp": datetime.now().isoformat(),
            "agent": agent_name,
            "task": task[:200],  # Truncate long tasks
            "result": result[:500],
            "duration_ms": duration_ms
        }
        
        self.traces.append(trace)
        
        # Write to disk
        with open(self.log_file, "a") as f:
            f.write(json.dumps(trace) + "\n")
    
    def analyze_traces(self) -> dict:
        """Analyze execution patterns."""
        
        if not self.traces:
            return {}
        
        by_agent = {}
        for trace in self.traces:
            agent = trace["agent"]
            if agent not in by_agent:
                by_agent[agent] = []
            by_agent[agent].append(trace)
        
        analysis = {}
        for agent, traces in by_agent.items():
            durations = [t["duration_ms"] for t in traces]
            analysis[agent] = {
                "total_executions": len(traces),
                "avg_duration_ms": sum(durations) / len(durations),
                "p95_duration_ms": sorted(durations)[int(len(durations) * 0.95)]
            }
        
        return analysis
```

---

## 6.4 Cost Optimization

### Agent Selection by Cost

```python
class CostAwareAgentSelector:
    """Select agents to minimize cost."""
    
    # Cost per execution (estimated)
    AGENT_COSTS = {
        "haiku_agent": 0.01,      # Cheap, fast
        "sonnet_agent": 0.05,     # Medium
        "opus_agent": 0.20        # Expensive, best quality
    }
    
    def select_agent_for_task(self, task: str, budget: float) -> str:
        """Select cheapest adequate agent for task."""
        
        # Easy tasks: use cheap agent
        if self._is_simple_task(task):
            return "haiku_agent"
        
        # Medium tasks: use medium agent if budget allows
        if budget >= self.AGENT_COSTS["sonnet_agent"]:
            return "sonnet_agent"
        
        # Complex tasks or unlimited budget: use best agent
        return "opus_agent"
    
    def _is_simple_task(self, task: str) -> bool:
        """Heuristic: is this a simple task?"""
        simple_keywords = ["summarize", "format", "list", "count"]
        return any(kw in task.lower() for kw in simple_keywords)
    
    def estimate_total_cost(self, tasks: list) -> float:
        """Estimate cost for multiple tasks."""
        
        total_cost = 0
        for task in tasks:
            agent = self.select_agent_for_task(task, float('inf'))
            total_cost += self.AGENT_COSTS[agent]
        
        return total_cost

selector = CostAwareAgentSelector()

# For a cheap task, use cheap agent
task = "Summarize this paragraph"
agent = selector.select_agent_for_task(task, budget=1.0)
print(agent)  # "haiku_agent"

# Estimate costs
tasks = ["summarize", "complex analysis", "code review"]
total = selector.estimate_total_cost(tasks)
print(f"Estimated cost: ${total:.2f}")
```

---

# Part 7: Interview Q&A

## 7.1 Core Concept Questions

### Q1: What Problem Does MCP Solve?

**Strong Answer:**
> MCP solves the tool integration fragmentation problem. Without MCP, every company that builds tools must create custom integrations with every LLM client. With MCP, tools expose a standardized interface, and LLM clients understand that one interface. This is similar to how USB solved the "every device has its own connector" problem. MCP makes tools composable, reusable, and language-agnostic.

---

### Q2: Explain MCP Architecture

**Strong Answer:**
> MCP has three layers: (1) LLM layer that needs external information, (2) MCP client that speaks the MCP protocol, (3) MCP servers that provide tools/resources. The flow: LLM says "I need to call get_weather", MCP client translates that to JSON-RPC and sends it to the appropriate MCP server, server executes and returns result, client feeds it back to LLM. The key innovation is standardization — the MCP protocol is the same regardless of what tool or LLM you're using.

---

### Q3: When Would You Use Multi-Agent vs Single Agent?

**Strong Answer:**
> Single agent is simpler and works when: the task fits within one agent's expertise, parallelization isn't needed, and you want minimal coordination overhead. Use multi-agent when: tasks naturally decompose (different specializations), you need parallelization (faster), you want robustness (if one agent fails, others continue), or different parts need deep expertise. Example: Analyzing a research paper is better with multi-agent (one for summarizing, one for finding gaps, one for criticizing) because each requires different expertise and can work in parallel.

---

## 7.2 Implementation Questions

### Q4: Design an MCP Server for a Database

**Strong Answer:**
> I'd build three components: (1) Tool definitions for queries (SELECT, INSERT, UPDATE), each with JSON schema for parameters. (2) Server loop that listens for MCP requests over stdio. (3) Safety layer that validates queries (prevent SQL injection) and enforces permissions. The flow: client sends `tools/call` with method name and parameters, server validates, executes against database, returns results. Key: parameterized queries to prevent injection, timeouts to prevent runaway queries, audit logging for all database access.

---

### Q5: How Would You Implement Manager-Worker Pattern?

**Strong Answer:**
> Manager-worker has three phases: (1) Manager decomposes the main task into subtasks (using LLM). (2) Manager assigns each subtask to the best-fitting worker based on expertise. (3) Workers execute in parallel, and manager synthesizes results. Implementation: manager is a specialized agent with decomposition logic, workers are specialized agents with domain expertise, coordination through a shared message queue or direct delegation. Example: analyzing a codebase — manager breaks it into "code review", "performance analysis", "security review", assigns each to specialized worker.

---

### Q6: How Do You Ensure Multi-Agent Systems Are Reliable?

**Strong Answer:**
> Five strategies: (1) Health monitoring — track each agent's success rate and latency, use healthy agents preferentially. (2) Fallback chains — if primary agent fails, route to backup agent. (3) Consensus checking — for critical decisions, have multiple agents vote. (4) Audit trails — log all agent decisions for debugging. (5) Graceful degradation — if some agents fail, system continues with reduced capability. Example: asking 3 different agents the same question, taking majority vote if confidence is low.

---

## 7.3 Architecture Questions

### Q7: Design a Multi-Agent System for Customer Support

**Strong Answer:**
> Three-tier system: (1) Router agent — classifies incoming ticket, decides if it's simple (FAQ), medium (needs investigation), or complex (escalate). (2) Specialist agents — one for billing, one for technical, one for general. (3) Escalation agent — routes complex issues to humans. Flow: ticket arrives → router classifies → if simple, FAQ agent answers directly → if medium, specialist agent researches and responds → if complex, flag for human. Monitoring: track resolution rate per specialist, alert if any agent has >10% failure rate. Cost optimization: use cheaper model for simple questions, better model for complex.

---

### Q8: How Would You Scale Multi-Agent System to 1000+ Concurrent Tasks?

**Strong Answer:**
> Four-part approach: (1) Agent pooling — maintain pools of agent instances (e.g., 10 copies of each specialist), load balance new tasks to idle agents. (2) Async execution — don't wait for agents sequentially; use async/await to handle multiple tasks. (3) Message queues — if all agents are busy, queue tasks (Redis, RabbitMQ) and process when agents become available. (4) Caching — cache common responses (FAQ answers, lookup data) to avoid calling agents for repeated requests. Monitoring: track queue depth and alert if backing up, track per-agent latency and adjust pool sizes dynamically.

---

*End of MCP + Multi-Agent Systems Master Guide*

**This guide covers everything from MCP protocol through production multi-agent systems.**

---

## 📚 Summary by Topic

### MCP (Parts 1-3)
- **Fundamentals:** What MCP is, why it matters, architecture
- **Building:** Implementing servers, tools, resources, prompts
- **Integration:** Claude + MCP, error handling, security

### Multi-Agent (Parts 4-6)
- **Concepts:** Agent types, roles, communication patterns
- **Architecture:** Manager-worker, hierarchical, collaborative, teams
- **Production:** Scalability, reliability, monitoring, cost optimization

### Key Takeaways
- ✅ MCP = standardized tool protocol (solves fragmentation)
- ✅ Multi-agent = specialized agents working together (solves complexity)
- ✅ Combine them = composable, scalable, robust AI systems
- ✅ Always think about: cost, reliability, monitoring

---

*Use this guide as your reference for building production AI systems with MCP and multiple agents.*
