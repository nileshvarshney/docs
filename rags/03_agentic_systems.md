# 🤖 Agentic Systems
## ReAct · Agentic Loops · LangGraph · MCP · Multi-Agent Orchestration

> **For:** Building autonomous, self-reasoning AI agents that plan, act, reflect, and coordinate.  
> **Assumed knowledge:** Tool calling (previous file), Python, LangChain/LangGraph.  
> **What you'll master:** Agent design patterns, state management, MCP servers, multi-agent systems.

---

## 📋 Quick Navigation
- [1. ReAct Pattern](#1-react-pattern)
- [2. Agentic Loops & State Management](#2-agentic-loops--state-management)
- [3. LangGraph Deep Dive](#3-langgraph-deep-dive)
- [4. Model Context Protocol (MCP)](#4-model-context-protocol-mcp)
- [5. Multi-Agent Orchestration](#5-multi-agent-orchestration)
- [6. Interview Q&A](#6-interview-qa)

---

# 1. ReAct Pattern

## 1.1 What Is ReAct?

**ReAct = Reasoning + Acting**

The model interleaves thinking and tool calls instead of deciding everything upfront. It observes the result, reasons about it, and decides the next action.

```
Traditional Flow:
User Input → [Plan Everything] → Execute Tools → Generate Answer

ReAct Flow:
User Input → [Think] → [Act] → [Observe] → [Think] → [Act] → ... → Answer
```

**Example:**

```
User: "What was the stock price of AAPL on Jan 1, 2024? Is it up from today?"

Traditional (BAD): Try to think of everything upfront → often wrong

ReAct (GOOD):
THINK: "I need to find AAPL's price on Jan 1, 2024, and today's price."
ACT: Call get_historical_price(AAPL, 2024-01-01)
OBSERVE: $185.64
THINK: "Got the historical price. Now I need today's price."
ACT: Call get_current_price(AAPL)
OBSERVE: $192.50
THINK: "Compare: $192.50 (today) vs $185.64 (Jan 1). That's an increase."
ANSWER: "AAPL was $185.64 on Jan 1, 2024. Today it's $192.50 (up 3.6%)."
```

## 1.2 ReAct System Prompt

```python
from anthropic import Anthropic

client = Anthropic()

REACT_SYSTEM_PROMPT = """You are an expert at reasoning through problems step-by-step.

When answering questions:
1. **THINK** — Analyze the question, identify what you need to know, plan your approach
2. **ACT** — Call a tool if you need information or to take action
3. **OBSERVE** — Read the tool result carefully
4. **THINK** — Based on the observation, what do you know now? What's next?
5. **REPEAT** until you have enough information to answer

Always show your reasoning. Format:

THINK: [Your analysis and next steps]
ACT: [Call a tool here, or answer if you have enough info]
OBSERVE: [The tool result — read it carefully]

Continue until you can provide a final answer with confidence."""

def react_agent(user_query: str, tools: list[dict]) -> str:
    """Simple ReAct agent using Anthropic SDK."""
    messages = [{"role": "user", "content": user_query}]

    for iteration in range(10):
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=2048,
            system=REACT_SYSTEM_PROMPT,
            tools=tools,
            messages=messages
        )

        # Check if we're done
        if response.stop_reason == "end_turn":
            # Extract final answer
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return ""

        # Collect tool calls and text
        tool_calls = [b for b in response.content if b.type == "tool_use"]
        text_blocks = [b for b in response.content if hasattr(b, "text")]

        if not tool_calls:
            # No tools called — return what we have
            return "\n".join(b.text for b in text_blocks if hasattr(b, "text"))

        # Add assistant response (text + tool calls)
        messages.append({"role": "assistant", "content": response.content})

        # Execute tools
        tool_results = []
        for tool_call in tool_calls:
            result = execute_tool(tool_call.name, tool_call.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tool_call.id,
                "content": result
            })

        # Add results back as OBSERVE step
        messages.append({"role": "user", "content": tool_results})

    return "Max iterations reached"
```

## 1.3 Key Insight: Explicit Reasoning

The model shows its work. You can audit it, detect errors, and build trust.

```
❌ Without ReAct:
Q: Is water wet?
A: Yes.
[Can't see the reasoning]

✅ With ReAct:
THINK: "The user asks if water is wet. 'Wet' means covered with water or liquid. Water is a liquid, so something made of water would be saturated with water — which matches the definition of wet."
ANSWER: Yes, water is wet. It's a liquid that covers and saturates surfaces, which is what "wet" means.
[Clear reasoning you can trust]
```

---

# 2. Agentic Loops & State Management

## 2.1 The Agent Loop Pattern

```python
class SimpleAgent:
    def __init__(self, model: str = "claude-opus-4-8"):
        self.model = model
        self.messages = []  # Conversation history
        self.max_iterations = 15

    def run(self, user_query: str, tools: list[dict]) -> str:
        """Execute the agent loop."""
        self.messages = [{"role": "user", "content": user_query}]

        for iteration in range(self.max_iterations):
            print(f"\n--- Iteration {iteration + 1} ---")

            # Step 1: Call LLM with tools
            response = client.messages.create(
                model=self.model,
                max_tokens=2048,
                tools=tools,
                messages=self.messages
            )

            # Step 2: Check termination
            if response.stop_reason == "end_turn":
                return self._extract_final_answer(response)

            # Step 3: Process tool calls
            tool_calls = [b for b in response.content if b.type == "tool_use"]

            if not tool_calls:
                return self._extract_final_answer(response)

            # Step 4: Update history
            self.messages.append({"role": "assistant", "content": response.content})

            # Step 5: Execute tools
            tool_results = []
            for tool_call in tool_calls:
                print(f"  → Calling {tool_call.name}({tool_call.input})")
                result = self._execute_tool(tool_call.name, tool_call.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": result
                })

            # Step 6: Feed results back
            self.messages.append({"role": "user", "content": tool_results})

        return "Max iterations reached"

    def _execute_tool(self, tool_name: str, tool_input: dict) -> str:
        # Your tool execution logic
        pass

    def _extract_final_answer(self, response) -> str:
        for block in response.content:
            if hasattr(block, "text"):
                return block.text
        return ""
```

## 2.2 State Management for Agents

Agents need to track **conversation state** — what happened, what tools succeeded, what failed.

```python
from typing import TypedDict, Annotated
from dataclasses import dataclass
from datetime import datetime
import operator

@dataclass
class ToolCallRecord:
    """Track each tool invocation."""
    iteration: int
    tool_name: str
    input: dict
    output: str
    duration_ms: float
    success: bool
    timestamp: datetime

class AgentState(TypedDict):
    """Shared state for the agent."""
    messages: Annotated[list, operator.add]  # Conversation history
    tool_calls: list[ToolCallRecord]  # All tools called (audit trail)
    current_iteration: int
    reasoning_chain: str  # The THINK-ACT-OBSERVE narrative
    final_answer: str
    started_at: datetime

class StatefulAgent:
    def __init__(self, model: str = "claude-opus-4-8"):
        self.model = model

    def run(self, user_query: str, tools: list[dict]) -> AgentState:
        """Execute agent with full state tracking."""
        state: AgentState = {
            "messages": [{"role": "user", "content": user_query}],
            "tool_calls": [],
            "current_iteration": 0,
            "reasoning_chain": "",
            "final_answer": "",
            "started_at": datetime.now()
        }

        for iteration in range(15):
            state["current_iteration"] = iteration

            response = client.messages.create(
                model=self.model,
                max_tokens=2048,
                tools=tools,
                messages=state["messages"]
            )

            # Extract reasoning
            reasoning = self._extract_thinking(response)
            state["reasoning_chain"] += f"\n[Iteration {iteration}]\n{reasoning}"

            if response.stop_reason == "end_turn":
                state["final_answer"] = self._extract_final_answer(response)
                return state

            tool_calls = [b for b in response.content if b.type == "tool_use"]

            if not tool_calls:
                state["final_answer"] = self._extract_final_answer(response)
                return state

            state["messages"].append({"role": "assistant", "content": response.content})

            # Execute tools with timing
            tool_results = []
            for tool_call in tool_calls:
                import time
                start = time.time()

                result = self._execute_tool(tool_call.name, tool_call.input)

                elapsed_ms = (time.time() - start) * 1000
                success = not isinstance(result, Exception)

                record = ToolCallRecord(
                    iteration=iteration,
                    tool_name=tool_call.name,
                    input=tool_call.input,
                    output=str(result),
                    duration_ms=elapsed_ms,
                    success=success,
                    timestamp=datetime.now()
                )
                state["tool_calls"].append(record)

                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": str(result)
                })

            state["messages"].append({"role": "user", "content": tool_results})

        state["final_answer"] = "Max iterations reached"
        return state

    def _extract_thinking(self, response) -> str:
        for block in response.content:
            if hasattr(block, "text"):
                return block.text
        return ""

    def _execute_tool(self, tool_name: str, tool_input: dict) -> str:
        # Implementation
        pass

    def _extract_final_answer(self, response) -> str:
        for block in response.content:
            if hasattr(block, "text"):
                return block.text
        return ""
```

---

# 3. LangGraph Deep Dive

## 3.1 What Is LangGraph?

LangGraph is a **state machine for agents**. You define:
1. **Nodes** — functions/LLMs that process state
2. **Edges** — transitions between nodes (deterministic or conditional)
3. **State** — shared data structure passed between nodes

```python
from langgraph.graph import START, END, StateGraph
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from typing import TypedDict, Annotated
import operator

# Define tools
@tool
def get_stock_price(symbol: str) -> str:
    """Get stock price."""
    return f"AAPL is $150.00" if symbol == "AAPL" else f"{symbol} not found"

@tool
def get_news(query: str) -> str:
    """Get news about a topic."""
    return f"Latest news on {query}: [news content]"

# Simple: Use prebuilt ReAct agent
def simple_react_agent():
    agent = create_react_agent(
        ChatAnthropic(model="claude-opus-4-8"),
        [get_stock_price, get_news]
    )
    return agent

# agent.invoke({"messages": [HumanMessage("What's AAPL price and latest news?")]})
```

## 3.2 Custom LangGraph Agent

```python
from langgraph.graph import StateGraph, START, END
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage
from langchain_core.tools import tool
from typing import TypedDict, Literal, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    current_tool: str
    iteration: int

# Tools
tools = [get_stock_price, get_news]
tool_map = {t.name: t for t in tools}

def model_node(state: AgentState) -> dict:
    """LLM decides what to do."""
    llm = ChatAnthropic(model="claude-opus-4-8")
    
    # Bind tools to LLM
    llm_with_tools = llm.bind_tools([get_stock_price, get_news])
    
    response = llm_with_tools.invoke(state["messages"])
    
    return {
        "messages": [response],
        "iteration": state.get("iteration", 0) + 1
    }

def tool_node(state: AgentState) -> dict:
    """Execute tool from last message."""
    last_message = state["messages"][-1]
    
    # Extract tool calls
    tool_calls = last_message.tool_calls
    tool_results = []
    
    for tool_call in tool_calls:
        tool_name = tool_call["name"]
        tool_input = tool_call["args"]
        
        tool = tool_map[tool_name]
        result = tool.invoke(tool_input)
        
        tool_results.append(ToolMessage(
            content=str(result),
            tool_call_id=tool_call["id"],
            name=tool_name
        ))
    
    return {"messages": tool_results}

def should_continue(state: AgentState) -> Literal["tools", END]:
    """Decide: continue with tools or end?"""
    last_message = state["messages"][-1]
    
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return END

# Build graph
graph = StateGraph(AgentState)

graph.add_node("model", model_node)
graph.add_node("tools", tool_node)

graph.add_edge(START, "model")
graph.add_conditional_edges(
    "model",
    should_continue,
    {
        "tools": "tools",
        END: END
    }
)
graph.add_edge("tools", "model")

agent = graph.compile()

# Run agent
result = agent.invoke({
    "messages": [HumanMessage("What's the stock price of AAPL?")],
    "iteration": 0
})
```

## 3.3 Persistent Graph Checkpointing

```python
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.graph import StateGraph, START, END

# Create a checkpoint (persistent memory)
checkpointer = SqliteSaver.from_conn_string(":memory:")  # or "file://db.sqlite"

graph = StateGraph(AgentState)
# ... add nodes and edges ...

# Compile with checkpointing
agent = graph.compile(checkpointer=checkpointer)

# Run with thread_id (enables resuming)
config = {"configurable": {"thread_id": "user_123"}}
result = agent.invoke({
    "messages": [HumanMessage("Start analyzing this data")],
    "iteration": 0
}, config=config)

# Later: resume the same thread
result = agent.invoke({
    "messages": [HumanMessage("Continue with step 2")],
    "iteration": 0
}, config=config)
# The agent remembers previous state!
```

---

# 4. Model Context Protocol (MCP)

## 4.1 What Is MCP?

MCP (Model Context Protocol) is **a standard for LLMs to request resources from external services.**

Think of it as: **A standardized way to build tool/resource integrations.**

```
┌─────────────────────────────────────┐
│          Your LLM (Claude)          │
│  "I need to search the knowledge   │
│   base for information about..."   │
└────────────────┬────────────────────┘
                 │
                 │ MCP Request
                 ↓
    ┌────────────────────────┐
    │   MCP Server (Yours)   │
    │  - Search tool         │
    │  - Database query      │
    │  - File access         │
    │  - API wrapper         │
    └────────────┬───────────┘
                 │
                 │ MCP Response
                 ↓
         ┌──────────────┐
         │ Tool Results │
         └──────────────┘
```

## 4.2 Building an MCP Server

```python
import json
from typing import Any, Callable
from dataclasses import dataclass

@dataclass
class MCPTool:
    name: str
    description: str
    input_schema: dict
    handler: Callable

class MCPServer:
    """Simple MCP server implementation."""

    def __init__(self, name: str, version: str):
        self.name = name
        self.version = version
        self.tools: dict[str, MCPTool] = {}

    def register_tool(
        self,
        name: str,
        description: str,
        input_schema: dict,
        handler: Callable
    ):
        """Register a tool with MCP."""
        self.tools[name] = MCPTool(
            name=name,
            description=description,
            input_schema=input_schema,
            handler=handler
        )

    def list_tools(self) -> list[dict]:
        """MCP endpoint: List available tools."""
        return [
            {
                "name": tool.name,
                "description": tool.description,
                "inputSchema": tool.input_schema
            }
            for tool in self.tools.values()
        ]

    def call_tool(self, tool_name: str, arguments: dict) -> str:
        """MCP endpoint: Execute a tool."""
        if tool_name not in self.tools:
            return json.dumps({"error": f"Unknown tool: {tool_name}"})

        tool = self.tools[tool_name]
        try:
            result = tool.handler(**arguments)
            return json.dumps({"result": result})
        except Exception as e:
            return json.dumps({"error": str(e)})


# Example: Knowledge Base MCP Server
mcp_server = MCPServer("KnowledgeBase", "1.0")

def search_kb(query: str, top_k: int = 5) -> list[dict]:
    """Search knowledge base (mock implementation)."""
    # In real impl: vector DB search, etc.
    return [
        {"id": "doc_1", "title": "RAG Patterns", "relevance": 0.95},
        {"id": "doc_2", "title": "Agent Design", "relevance": 0.87}
    ]

def get_document(doc_id: str) -> dict:
    """Get full document content."""
    docs = {
        "doc_1": {"title": "RAG Patterns", "content": "..."},
        "doc_2": {"title": "Agent Design", "content": "..."}
    }
    return docs.get(doc_id, {"error": "Not found"})

# Register tools
mcp_server.register_tool(
    name="search_knowledge_base",
    description="Search internal docs by keyword or semantic similarity",
    input_schema={
        "type": "object",
        "properties": {
            "query": {"type": "string"},
            "top_k": {"type": "integer", "default": 5}
        },
        "required": ["query"]
    },
    handler=search_kb
)

mcp_server.register_tool(
    name="get_document",
    description="Retrieve full content of a document by ID",
    input_schema={
        "type": "object",
        "properties": {
            "doc_id": {"type": "string"}
        },
        "required": ["doc_id"]
    },
    handler=get_document
)

# List available tools
tools = mcp_server.list_tools()
print(json.dumps(tools, indent=2))

# Execute a tool
result = mcp_server.call_tool("search_knowledge_base", {"query": "RAG agents"})
print(result)
```

## 4.3 Integrating MCP with Claude

```python
# When Claude calls a tool defined in your MCP server,
# the response is transparent to the model

def agent_with_mcp(user_query: str):
    """Agent using MCP tools."""
    mcp_tools = mcp_server.list_tools()

    # Convert MCP tools to Claude format
    claude_tools = [
        {
            "name": tool["name"],
            "description": tool["description"],
            "input_schema": tool["inputSchema"]
        }
        for tool in mcp_tools
    ]

    messages = [{"role": "user", "content": user_query}]

    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=2048,
        tools=claude_tools,
        messages=messages
    )

    # Process tool calls
    tool_calls = [b for b in response.content if b.type == "tool_use"]

    if not tool_calls:
        return response.content[0].text

    messages.append({"role": "assistant", "content": response.content})

    # Execute via MCP server
    tool_results = []
    for tool_call in tool_calls:
        result = mcp_server.call_tool(tool_call.name, tool_call.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": tool_call.id,
            "content": result
        })

    messages.append({"role": "user", "content": tool_results})

    # Final response
    final = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=2048,
        tools=claude_tools,
        messages=messages
    )

    for b in final.content:
        if hasattr(b, "text"):
            return b.text
```

---

# 5. Multi-Agent Orchestration

## 5.1 Manager-Worker Pattern

One "manager" agent routes tasks to specialized "worker" agents.

```python
class SpecializedAgent:
    def __init__(self, name: str, role: str, tools: list[dict]):
        self.name = name
        self.role = role
        self.tools = tools

    def run(self, task: str) -> str:
        """Execute task using this agent's specialized tools."""
        messages = [{"role": "user", "content": task}]

        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            system=f"You are {self.role}. Use your specialized tools to complete tasks.",
            tools=self.tools,
            messages=messages
        )

        # Simple: return response (in real impl, handle tool calls)
        for b in response.content:
            if hasattr(b, "text"):
                return b.text
        return ""

class ManagerAgent:
    def __init__(self):
        self.workers = {}

    def register_worker(self, name: str, agent: SpecializedAgent):
        self.workers[name] = agent

    def orchestrate(self, user_query: str) -> str:
        """Route user query to appropriate worker(s)."""
        # Step 1: Classify what type of task
        classification = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=100,
            messages=[{
                "role": "user",
                "content": f"""Classify this task:
"{user_query}"

Choose ONE: research, code, writing, analysis, other"""
            }]
        )

        task_type = classification.content[0].text.strip().lower().split(':')[0]

        # Step 2: Route to worker
        if "code" in task_type:
            worker = self.workers.get("coder")
        elif "research" in task_type:
            worker = self.workers.get("researcher")
        elif "write" in task_type:
            worker = self.workers.get("writer")
        else:
            worker = self.workers.get("general")

        if not worker:
            return "No suitable worker found"

        # Step 3: Execute with worker
        result = worker.run(user_query)

        # Step 4: Manager reviews and refines if needed
        final = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": f"""Review and polish this response:
{result}

Ensure quality, clarity, and completeness."""
            }]
        )

        return final.content[0].text


# Setup
manager = ManagerAgent()

coder_agent = SpecializedAgent(
    name="coder",
    role="expert Python developer",
    tools=[write_code_tool, debug_tool, run_tests_tool]
)

researcher_agent = SpecializedAgent(
    name="researcher",
    role="research expert",
    tools=[search_papers_tool, analyze_data_tool]
)

writer_agent = SpecializedAgent(
    name="writer",
    role="professional writer",
    tools=[outline_tool, edit_tool]
)

manager.register_worker("coder", coder_agent)
manager.register_worker("researcher", researcher_agent)
manager.register_worker("writer", writer_agent)

# Run
result = manager.orchestrate("Write a Python script that analyzes sentiment from CSV files")
```

## 5.2 Collaborative Multi-Agent (Everyone Contributes)

```python
class CollaborativeAgent:
    def __init__(self, name: str, role: str):
        self.name = name
        self.role = role

    def contribute(self, problem: str, context: str) -> str:
        """Contribute perspective on the problem."""
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=512,
            messages=[{
                "role": "user",
                "content": f"""You are {self.role}.
Problem: {problem}

Previous thoughts:
{context}

Add your perspective (2-3 sentences max). Be concise."""
            }]
        )
        return response.content[0].text

def collaborative_brainstorm(problem: str) -> str:
    """Multiple agents collaborate on a solution."""
    agents = [
        CollaborativeAgent("strategist", "a strategic thinker"),
        CollaborativeAgent("engineer", "a technical engineer"),
        CollaborativeAgent("designer", "a UX designer"),
        CollaborativeAgent("business", "a business analyst")
    ]

    context = ""

    for agent in agents:
        contribution = agent.contribute(problem, context)
        context += f"\n{agent.name}: {contribution}"

    # Synthesize
    synthesis = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"""Synthesize these perspectives into a unified solution:
{context}

Create a cohesive plan that incorporates insights from each perspective."""
        }]
    )

    return synthesis.content[0].text


result = collaborative_brainstorm("How should we design a new chat interface?")
print(result)
```

---

# 6. Interview Q&A

## Q1: ReAct vs Standard Agent Loop — When Does ReAct Help?

**Strong Answer:**
> ReAct (showing reasoning explicitly) helps when:
> 1. **Complex multi-step reasoning** — The model benefits from thinking out loud before acting.
> 2. **Debugging needed** — You need to audit why the agent made a decision.
> 3. **User education** — You want to show users the agent's reasoning.
>
> It HURTS when:
> 1. **Simple lookup tasks** — "Get me the stock price of AAPL" doesn't need reasoning.
> 2. **Latency is critical** — Extra THINK→ACT→OBSERVE tokens add latency.
> 3. **Token budget is tight** — Reasoning tokens = $$$.
>
> **Rule:** Use ReAct for reasoning-heavy tasks, plain tool-use for simple lookups.

---

## Q2: How Do You Prevent Agent Loops (Calling Same Tool Forever)?

**Strong Answer:**
> Three safeguards:
> 1. **max_iterations limit** (typically 10–15). Beyond that, stop and return what you have.
> 2. **Detect repeated calls** — If the agent calls the same tool with the same input 3+ times, break the loop and inform it this isn't working.
> 3. **Tool error messages** — When a tool fails, return a clear error so the model adapts instead of retrying:
>    ```python
>    if tool_result is_error:
>        return f"Tool {tool_name} failed: {error_reason}. Try a different approach."
>    ```
>
> **Prevention at design time:**
> - Clear tool descriptions help the model pick the right tool
> - Explicit error messages prevent "retry the same thing" behavior
> - System prompt: "If a tool call fails, try a different approach, don't retry the same call."

---

## Q3: LangGraph vs Manual Agent Loop — When to Use Each?

**Strong Answer:**
> **Manual Loop (Low-level control):**
> Pros: Total control, easy to debug, understand every step  
> Cons: Boilerplate, error handling, state management is your job  
> Use when: You need custom logic, unusual workflows, teaching/learning
>
> **LangGraph (Higher-level abstraction):**
> Pros: Built-in state management, checkpointing, streaming, cleaner code  
> Cons: Less control, harder to debug if something goes wrong, learning curve  
> Use when: Standard agent patterns, production systems, need persistence
>
> **My recommendation:** Start with manual loops to understand the mechanics, then move to LangGraph for production to reduce bugs and maintenance.

---

## Q4: Persistent Agent State — When Do You Need Checkpointing?

**Strong Answer:**
> You need checkpointing when:
> 1. **Multi-turn conversations** — User returns later, agent resumes context
> 2. **Long-running tasks** — Agent takes 30+ seconds; user might disconnect/reconnect
> 3. **Debugging** — Replay exact execution path after a failure
> 4. **Cost tracking** — Persist which tools were called for billing
>
> Example: Chatbot with memory
> ```
> Session 1: User asks "Analyze this CSV"
>           Agent calls read_file, analyze_tool, returns insights
>           State is checkpointed
>
> Session 2 (later): User asks "What did you find about column X?"
>                   Agent resumes from checkpoint
>                   Remembers previous analysis without re-running tools
> ```
>
> Use SQLiteSaver (simple) or PostgresSaver (production).

---

## Q5: MCP Servers — How Is This Different from Just Passing Tool Definitions?

**Strong Answer:**
> **Tool Definitions (Claude native):**
> - You define tools in JSON schema
> - Send them to Claude every request
> - Claude calls them, you execute
> - Tightly coupled — Claude and your code are in same process (usually)
>
> **MCP Servers:**
> - Standardized protocol for tool discovery and execution
> - Server is separate process/service
> - Any MCP-compatible client (Claude, other LLMs) can use your tools
> - You define tools once, any client can call them
> - Better for: multi-client scenarios, sharing tools across teams
>
> **Analogy:**
> - Tool Definitions = shipping your code in the package
> - MCP = opening a store; multiple customers can come call tools
>
> Use MCP when you want tools accessible to multiple AI systems or clients. Use native tool definitions for simple single-agent setups.

---

## Q6: Multi-Agent Orchestration — Manager-Worker vs Collaborative?

**Strong Answer:**
> **Manager-Worker:**
> - One master agent routes to specialists
> - Good for: specialized tasks, clear routing logic
> - Example: "Is this a bug fix or feature request? → Route to bug-handler or feature-handler"
> - Pro: Clear separation, easy to scale workers
> - Con: Manager bottleneck, single point of failure
>
> **Collaborative:**
> - All agents contribute perspective, then synthesize
> - Good for: brainstorming, complex decisions needing multiple viewpoints
> - Example: "Design this product → get input from eng, design, business"
> - Pro: More creative, catches blind spots
> - Con: Slower (sequential), harder to coordinate
>
> **Choice:** Manager-Worker for task routing. Collaborative for brainstorming/analysis.

---

## Q7: How Do You Handle Agent Hallucinations?

**Strong Answer:**
> Agents hallucinate when tools aren't available or the model thinks it can do something it can't.
>
> **Prevention:**
> 1. **Clear tool definitions** — Specify exactly what each tool does, when to use it
> 2. **System prompt** — "Only use the provided tools. Don't claim you can do things you can't"
> 3. **No "thinking" tools** — Don't give the agent a tool to hallucinate outputs
>
> **Detection:**
> 1. **Verify tool results** — After agent calls a tool, check: did it actually execute? Did it work?
> 2. **Output validation** — Parse agent output, check it matches expected format
> 3. **Confidence scoring** — Ask model how confident it is; if <0.6, flag for review
>
> **Recovery:**
> ```python
> if agent_output_is_suspicious(agent_output):
>     return "I'm not fully confident in that answer. Let me reconsider."
>     # Run again, or ask user for clarification
> ```

---

## Q8: Streaming Agent Outputs — Best Practices?

**Strong Answer:**
> **When to stream:**
> - Long-running agents (5+ second execution)
> - User experience: show progress ("Thinking...", "Calling tools...")
> - Intermediate results available before final answer
>
> **What to stream:**
> 1. Reasoning tokens (THINK blocks)
> 2. Tool names as they're called
> 3. Tool results (if long)
> 4. Final answer
>
> **Implementation:**
> ```python
> with client.messages.stream(...) as stream:
>     for event in stream:
>         if "tool_use" in str(event):
>             print(f"Calling tool...")
>         elif "text_delta" in str(event):
>             print(event.delta.text, end="", flush=True)
> ```
>
> **Caution:** Streaming adds complexity. Only do it if latency matters to your UX.

---

*End of Agentic Systems Master Guide*

**Next:** File 4 — Production Deployment & Evaluation

