# 🛠️ Tool Use & Function Calling
## Complete Master Guide — Everything in One Document

> **For:** Senior engineers building agentic systems, production agents, and LLM systems that interact with external tools.  
> **Assumed knowledge:** Python, LangChain/LangGraph basics, prompt engineering fundamentals.  
> **What you'll own:** Designing tool schemas, implementing tool selection logic, handling errors, streaming with tools, building production-grade tool pipelines.

---

## 📋 Quick Navigation
- [1. Mental Model & Core Concepts](#1-mental-model--core-concepts)
- [2. Tool Schema Design](#2-tool-schema-design)
- [3. Implementation Patterns](#3-implementation-patterns)
- [4. Advanced Patterns](#4-advanced-patterns)
- [5. Production Reliability](#5-production-reliability)
- [6. Interview Q&A](#6-interview-qa)

---

# 1. Mental Model & Core Concepts

## 1.1 What Is Tool Use?

Tool use (aka function calling) is **the mechanism by which LLMs request execution of external code.**

The model:
1. Decides a tool might help (understanding, calculation, API call, database query)
2. Specifies which tool and what arguments
3. You execute that tool
4. Feed the result back to the model as context
5. Model continues reasoning with the result

```
┌─────────────────────────────────────────────────────────────┐
│ User: "What's the weather in San Francisco tomorrow?"      │
├─────────────────────────────────────────────────────────────┤
│ Model (reasoning): "I need weather data. I'll use the       │
│                    get_weather tool with location='SF'      │
│                    and date='tomorrow'"                     │
├─────────────────────────────────────────────────────────────┤
│ You (executor): Calls `get_weather('SF', 'tomorrow')`      │
│                 Gets result: {"temp": 72, "condition": ...} │
├─────────────────────────────────────────────────────────────┤
│ Model (with result as context): "Based on this weather     │
│                                  data, it will be 72°F      │
│                                  and sunny tomorrow."        │
└─────────────────────────────────────────────────────────────┘
```

## 1.2 Why This Matters

**Without tool use:** Model can't update knowledge, do real calculations, or trigger actions.
```python
Q: "What files are in /tmp?"
A: "I don't know — my training data doesn't include your filesystem."
```

**With tool use:** Model bridges the gap between reasoning and execution.
```python
Q: "What files are in /tmp?"
Model: Calls list_files("/tmp")
A: "Your /tmp contains: cache.db, temp_logs, uploads/ [actual answer]"
```

This is the foundation of **autonomous agents** — the next section in your learning path.

---

## 1.3 Tool Use Workflow (The Loop)

```
┌──────────────────────────────────────┐
│  1. You call model with tools        │
│     defined + user message           │
└────────────┬─────────────────────────┘
             ↓
     ┌───────────────────────┐
     │ 2. Model reasons:     │
     │    Do I need a tool?  │
     └────────┬──────────────┘
              ↓ YES
     ┌───────────────────────────────┐
     │ 3. Model outputs:             │
     │  {tool: "name", args: {...}}  │
     └────────┬──────────────────────┘
              ↓
     ┌───────────────────────────────┐
     │ 4. You execute the tool       │
     │    (call your function)        │
     └────────┬──────────────────────┘
              ↓
     ┌───────────────────────────────┐
     │ 5. Tool result returned        │
     │    to model as context         │
     └────────┬──────────────────────┘
              ↓ Continue reasoning
     ┌───────────────────────────────┐
     │ 6. Model continues,            │
     │    may call another tool       │
     │    or give final answer        │
     └───────────────────────────────┘
```

The model **never** executes code directly. You always execute and return results.

---

## 1.4 Key Differences: Claude vs OpenAI Tool Calling

### Claude (Anthropic) Approach

```python
# Tools are passed as a dedicated parameter
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=[...],  # Separate tools array
    messages=[...]
)

# Tool calls appear in response.content as block type: tool_use
for block in response.content:
    if block.type == "tool_use":
        tool_name = block.name
        tool_input = block.input  # dict
        tool_id = block.id  # For matching results
```

**Advantages:**
- Clear separation: tools ≠ function definitions
- Native support for parallel tool calls (multiple tools in one response)
- Streaming tool calls (see tool invocation before result)

### OpenAI Approach

```python
# Tools are "functions" in a special schema
response = client.chat.completions.create(
    model="gpt-4o",
    tools=[
        {
            "type": "function",
            "function": {
                "name": "get_weather",
                "description": "...",
                "parameters": {...}
            }
        }
    ],
    messages=[...]
)

# Tool calls appear in message.tool_calls (separate from message.content)
for tool_call in response.message.tool_calls:
    tool_name = tool_call.function.name
    tool_input = json.loads(tool_call.function.arguments)  # String → dict
```

**Differences:**
- Tools wrapped in `{"type": "function", "function": {...}}`
- Tool call arguments are JSON **strings**, not dicts (must parse)
- Parallel tool support via list of `tool_calls`

**For this guide: We use Claude (Anthropic) as primary, with OpenAI patterns noted where different.**

---

# 2. Tool Schema Design

## 2.1 The JSON Schema Format (Claude)

Tools are defined with **JSON Schema** — a formal specification for tool inputs.

```python
from anthropic import Anthropic

client = Anthropic()

# Well-designed tool schema
tools = [
    {
        "name": "get_user_by_id",
        "description": "Fetch a user profile by numeric ID from the database. Use when you need user information like name, email, signup date, or subscription status.",
        "input_schema": {
            "type": "object",
            "properties": {
                "user_id": {
                    "type": "integer",
                    "description": "The unique numeric ID of the user (e.g., 12345). Always an integer, never a string."
                }
            },
            "required": ["user_id"]
        }
    },
    {
        "name": "update_user_profile",
        "description": "Update a user's profile information (email, phone, preferences). Changes are immediately persisted.",
        "input_schema": {
            "type": "object",
            "properties": {
                "user_id": {
                    "type": "integer",
                    "description": "User ID to update"
                },
                "email": {
                    "type": ["string", "null"],
                    "description": "New email address (RFC 5322 format). Set to null to leave unchanged."
                },
                "phone": {
                    "type": ["string", "null"],
                    "description": "New phone number in E.164 format (+1234567890). Set to null to leave unchanged."
                },
                "newsletter_opt_in": {
                    "type": "boolean",
                    "description": "Opt-in to weekly newsletter"
                }
            },
            "required": ["user_id"]
        }
    }
]

response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "Get user 12345's email address"}]
)
```

## 2.2 Best Practices for Tool Schemas

### ❌ Anti-Pattern: Vague Descriptions

```python
# Bad: Model has no idea what this does
{
    "name": "process_data",
    "description": "Process data",
    "input_schema": {
        "type": "object",
        "properties": {
            "data": {"type": "string"},
            "mode": {"type": "string"}
        }
    }
}
```

### ✅ Pattern: Specific, Actionable Descriptions

```python
# Good: Model knows exactly when and how to use this
{
    "name": "calculate_subscription_renewal_date",
    "description": "Calculate when a user's subscription will renew based on their signup date and plan duration. Use this when answering questions about renewal dates or upcoming payments.",
    "input_schema": {
        "type": "object",
        "properties": {
            "user_id": {
                "type": "integer",
                "description": "Unique user ID (numeric, from the users table)"
            },
            "plan_type": {
                "type": "string",
                "enum": ["monthly", "annual", "lifetime"],
                "description": "Subscription plan type. Always one of: monthly (renews monthly), annual (renews yearly), lifetime (never renews)"
            }
        },
        "required": ["user_id", "plan_type"]
    }
}
```

### Key Rules for Schemas

| Rule | Why | Example |
|------|-----|---------|
| **Specific types** | So the model knows valid input format | `"type": "integer"` not `"type": "string"` for IDs |
| **Enums for choices** | Forces valid values; prevents invalid arguments | `"enum": ["pending", "approved", "rejected"]` |
| **Mark required fields** | Model knows what's mandatory | `"required": ["user_id"]` |
| **Detailed descriptions** | Model learns when/how to use the tool | "Used to fetch user profiles from the database" |
| **Type coercion hints** | Help for common mistakes | "Always send as ISO 8601 date: YYYY-MM-DD" |
| **Nullable fields** | Allow "no change" option for updates | `"type": ["string", "null"]` for optional updates |

---

## 2.3 Designing Tool Granularity

**Question: Should `search_users` be one tool or many tools (search_by_email, search_by_name, search_by_id)?**

### Approach 1: One Coarse-Grained Tool (Flexible)

```python
{
    "name": "search_users",
    "description": "Search users by multiple criteria. Can search by email, name, phone, or ID.",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "The search term or value (email, name fragment, phone, or numeric ID)"
            },
            "search_type": {
                "type": "string",
                "enum": ["email", "name", "phone", "id"],
                "description": "Type of search to perform"
            },
            "limit": {
                "type": "integer",
                "default": 10,
                "description": "Max results to return (1-100)"
            }
        },
        "required": ["query", "search_type"]
    }
}
```

**Pros:** Flexible, fewer tools to define, model picks the right search_type.  
**Cons:** Larger schema, more room for model error on search_type.

### Approach 2: Multiple Fine-Grained Tools (Clear)

```python
tools = [
    {
        "name": "get_user_by_id",
        "description": "Get exact user by numeric ID",
        "input_schema": {...}
    },
    {
        "name": "search_users_by_email",
        "description": "Search users by email address",
        "input_schema": {...}
    },
    {
        "name": "search_users_by_name",
        "description": "Search users by full or partial name",
        "input_schema": {...}
    }
]
```

**Pros:** Clear intent, less ambiguity, easier to control/log.  
**Cons:** More tools to maintain, more potential for tool selection errors.

### Recommendation

**Start with fine-grained tools (Approach 2).** It's easier to monitor which tools are being called, and the model makes fewer mistakes when tools are orthogonal. Combine only if the cost of managing many tools outweighs the clarity benefit.

---

## 2.4 Using Pydantic for Schema Validation

```python
from pydantic import BaseModel, Field
from typing import Optional, Literal
import json

# Define tools as Pydantic models (clean, validated)
class GetUserRequest(BaseModel):
    user_id: int = Field(..., description="Unique numeric user ID")
    include_metadata: bool = Field(
        default=False,
        description="Include internal metadata (admin only)"
    )

class UpdateUserRequest(BaseModel):
    user_id: int
    email: Optional[str] = Field(None, description="New email (RFC 5322)")
    phone: Optional[str] = Field(None, description="Phone in E.164 format")
    newsletter_opt_in: Optional[bool] = None

# Convert Pydantic model to Claude tool schema
def pydantic_to_claude_tool(model: type[BaseModel], name: str, description: str) -> dict:
    schema = model.model_json_schema()
    return {
        "name": name,
        "description": description,
        "input_schema": {
            "type": "object",
            "properties": schema.get("properties", {}),
            "required": schema.get("required", [])
        }
    }

tools = [
    pydantic_to_claude_tool(
        GetUserRequest,
        "get_user",
        "Fetch user profile by ID"
    ),
    pydantic_to_claude_tool(
        UpdateUserRequest,
        "update_user",
        "Update user profile fields"
    )
]

# When model calls a tool, validate the input
def execute_tool(tool_name: str, tool_input: dict) -> str:
    try:
        if tool_name == "get_user":
            req = GetUserRequest(**tool_input)  # Pydantic validates
            return get_user_from_db(req.user_id)
        elif tool_name == "update_user":
            req = UpdateUserRequest(**tool_input)
            return update_user_in_db(req)
    except ValueError as e:
        return f"Invalid input: {e}"
```

---

# 3. Implementation Patterns

## 3.1 Basic Tool Use Loop (Anthropic SDK)

```python
from anthropic import Anthropic

client = Anthropic()

def get_stock_price(symbol: str) -> str:
    """Simulate fetching stock price."""
    prices = {"AAPL": "$150.00", "GOOGL": "$140.00", "MSFT": "$380.00"}
    return prices.get(symbol, "Symbol not found")

def get_company_info(symbol: str) -> str:
    """Simulate fetching company info."""
    info = {
        "AAPL": "Apple Inc. - Technology company",
        "GOOGL": "Alphabet Inc. - Search and advertising"
    }
    return info.get(symbol, "Company not found")

# Define tools
tools = [
    {
        "name": "get_stock_price",
        "description": "Get the current stock price for a given ticker symbol",
        "input_schema": {
            "type": "object",
            "properties": {
                "symbol": {
                    "type": "string",
                    "description": "Stock ticker symbol (e.g., AAPL, GOOGL)"
                }
            },
            "required": ["symbol"]
        }
    },
    {
        "name": "get_company_info",
        "description": "Get company information by ticker symbol",
        "input_schema": {
            "type": "object",
            "properties": {
                "symbol": {
                    "type": "string",
                    "description": "Stock ticker symbol"
                }
            },
            "required": ["symbol"]
        }
    }
]

def process_tool_call(tool_name: str, tool_input: dict) -> str:
    """Execute a tool call and return result."""
    if tool_name == "get_stock_price":
        return get_stock_price(tool_input["symbol"])
    elif tool_name == "get_company_info":
        return get_company_info(tool_input["symbol"])
    else:
        return f"Unknown tool: {tool_name}"

def agent_loop(user_message: str, max_iterations: int = 10) -> str:
    """Run the agent loop with tool use."""
    messages = [{"role": "user", "content": user_message}]

    for iteration in range(max_iterations):
        # Call Claude with tools
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        # Check stop reason
        if response.stop_reason == "end_turn":
            # Model finished — extract final text response
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return "No response generated"

        # Process tool calls
        tool_calls = [block for block in response.content if block.type == "tool_use"]
        text_blocks = [block for block in response.content if hasattr(block, "text")]

        if not tool_calls:
            # No tools called but stop_reason != end_turn → shouldn't happen, but handle it
            return response.content[0].text if response.content else "No response"

        # Add assistant's response to message history
        messages.append({
            "role": "assistant",
            "content": response.content  # Includes both text and tool_use blocks
        })

        # Execute each tool call and collect results
        tool_results = []
        for tool_call in tool_calls:
            tool_result = process_tool_call(tool_call.name, tool_call.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tool_call.id,
                "content": tool_result
            })

        # Add tool results back to messages
        messages.append({
            "role": "user",
            "content": tool_results
        })

        print(f"Iteration {iteration + 1}: Called {len(tool_calls)} tool(s)")

    return "Max iterations reached without final response"

# Test the agent
result = agent_loop("What is the stock price of AAPL and tell me about Google?")
print(result)
```

**Output:**
```
Iteration 1: Called 2 tool(s)
Stock price for AAPL is $150.00.

Alphabet Inc. (GOOGL) is a search and advertising company. 
Their current stock price is $140.00.
```

---

## 3.2 Parallel Tool Calls

The model can request multiple tools in a single turn.

```python
def agent_loop_with_parallel_tools(user_message: str) -> str:
    """Same as before, but the model naturally calls multiple tools."""
    messages = [{"role": "user", "content": user_message}]

    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    # Collect all tool calls from this response
    tool_calls = [block for block in response.content if block.type == "tool_use"]

    if not tool_calls:
        # No tools — return final response
        return response.content[0].text if response.content else ""

    messages.append({"role": "assistant", "content": response.content})

    # Execute ALL tools in parallel (they don't depend on each other)
    # In async, this would be: await asyncio.gather(*[execute(tc) for tc in tool_calls])
    tool_results = []
    for tool_call in tool_calls:
        result = process_tool_call(tool_call.name, tool_call.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": tool_call.id,
            "content": result
        })

    messages.append({"role": "user", "content": tool_results})

    # Final response
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    for block in response.content:
        if hasattr(block, "text"):
            return block.text

# Test parallel tool calls
result = agent_loop_with_parallel_tools("Get the stock price and info for AAPL, GOOGL, and MSFT")
print(result)
```

**Advantage:** Much faster when tools are independent (no waiting for sequential execution).

---

## 3.3 With LangChain (Higher-Level Abstraction)

```python
from langchain_anthropic import ChatAnthropic
from langchain.agents import initialize_agent, Tool
from langchain.tools import tool

# Define tools using @tool decorator
@tool
def get_stock_price(symbol: str) -> str:
    """Get the current stock price for a ticker symbol."""
    prices = {"AAPL": "$150.00", "GOOGL": "$140.00"}
    return prices.get(symbol, "Symbol not found")

@tool
def get_company_info(symbol: str) -> str:
    """Get company information by ticker."""
    info = {"AAPL": "Apple Inc. - Technology", "GOOGL": "Alphabet Inc. - Search"}
    return info.get(symbol, "Company not found")

# Convert to LangChain Tool format
tools = [
    Tool.from_function(get_stock_price),
    Tool.from_function(get_company_info)
]

# Create agent
llm = ChatAnthropic(model="claude-opus-4-8")
agent = initialize_agent(
    tools,
    llm,
    agent="tool-using-react-agent",
    verbose=True
)

# Run agent
result = agent.run("What's the stock price of AAPL and info on Google?")
print(result)
```

**LangChain handles:**
- Tool formatting into Claude schemas
- Tool call extraction from response
- Tool execution and result feeding
- Agentic loop management

**Trade-off:** Less control, but much faster to prototype.

---

## 3.4 With LangGraph (Stateful Agents)

```python
from langgraph.graph import START, END, StateGraph
from langgraph.prebuilt import create_react_agent
from langchain_anthropic import ChatAnthropic
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]  # Accumulate messages
    current_tool: str
    tool_count: int

# Tools (same as before)
def get_stock_price(symbol: str) -> str:
    prices = {"AAPL": "$150.00", "GOOGL": "$140.00"}
    return prices.get(symbol, "Not found")

def get_company_info(symbol: str) -> str:
    info = {"AAPL": "Apple Inc.", "GOOGL": "Alphabet Inc."}
    return info.get(symbol, "Not found")

# Simple approach: Use create_react_agent (abstraction)
agent = create_react_agent(
    ChatAnthropic(model="claude-opus-4-8"),
    [
        Tool.from_function(get_stock_price),
        Tool.from_function(get_company_info)
    ]
)

# This is a LangGraph that handles the full agentic loop
# Can be invoked, streamed, or embedded in larger workflows

# Advanced approach: Custom graph for more control
def tool_node(state: AgentState):
    """Custom tool execution node."""
    messages = state["messages"]
    # Extract and execute tools from last message
    # ... implementation details ...
    return {"messages": [...], "tool_count": state["tool_count"] + 1}

def model_node(state: AgentState):
    """LLM reasoning node."""
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

graph = StateGraph(AgentState)
graph.add_node("model", model_node)
graph.add_node("tools", tool_node)

graph.add_edge(START, "model")
graph.add_conditional_edges(
    "model",
    lambda x: "tools" if tool_calls_in_response(x) else END
)
graph.add_edge("tools", "model")

agent = graph.compile()
```

---

# 4. Advanced Patterns

## 4.1 Tool Chaining & Dependency Management

When tools depend on each other's output.

```python
def calculate_retirement_plan(user_id: int) -> dict:
    """
    Complex multi-step calculation that chains tool calls.
    
    Flow:
    1. Get user profile (age, salary)
    2. Get investment portfolio (current balance)
    3. Calculate years to retirement
    4. Project retirement savings
    """
    
    # Step 1: Get user data
    user = execute_tool("get_user", {"user_id": user_id})
    age = user["age"]
    annual_salary = user["annual_salary"]

    # Step 2: Get investment data (depends on step 1)
    portfolio = execute_tool("get_investment_portfolio", {"user_id": user_id})
    current_balance = portfolio["total_value"]

    # Step 3: Perform local calculation
    years_to_retirement = max(0, 67 - age)  # Retirement age 67
    annual_contribution = annual_salary * 0.10  # Assume 10% savings rate

    # Step 4: Get projection from tool (for complex math)
    projection = execute_tool("project_savings", {
        "current_balance": current_balance,
        "annual_contribution": annual_contribution,
        "years": years_to_retirement,
        "annual_return": 0.07  # 7% average return
    })

    return {
        "user_id": user_id,
        "age": age,
        "current_balance": current_balance,
        "years_to_retirement": years_to_retirement,
        "projected_retirement_savings": projection["final_balance"],
        "monthly_retirement_income": projection["final_balance"] / (25 * 12)  # 25-year withdrawal
    }
```

**Problem:** Complex logic chains become hard to trace. In agents, this is handled naturally because the model decides the order.

---

## 4.2 Tool Error Handling & Recovery

```python
def safe_tool_execution(
    tool_name: str,
    tool_input: dict,
    max_retries: int = 2
) -> tuple[bool, str]:
    """
    Execute a tool with error handling and retry logic.
    Returns: (success: bool, result: str)
    """
    for attempt in range(max_retries):
        try:
            if tool_name == "get_stock_price":
                symbol = tool_input.get("symbol", "").upper()
                if not symbol:
                    raise ValueError("Symbol is required")
                if len(symbol) > 5:
                    raise ValueError(f"Invalid symbol: {symbol}")
                # Call API / database
                result = fetch_stock_price(symbol)
                return True, result

            elif tool_name == "transfer_funds":
                amount = tool_input.get("amount")
                if not isinstance(amount, (int, float)) or amount <= 0:
                    raise ValueError("Amount must be positive number")
                # Attempt transfer
                result = process_transfer(amount, tool_input)
                return True, result

        except ValueError as e:
            # Invalid input — no retry
            return False, f"Input error: {e}"
        except ConnectionError as e:
            # Transient error — retry
            if attempt < max_retries - 1:
                print(f"Connection error (attempt {attempt + 1}), retrying...")
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            return False, f"Connection failed after {max_retries} attempts: {e}"
        except Exception as e:
            # Unexpected error
            return False, f"Tool execution failed: {e}"

    return False, "Unknown error"

def agent_loop_with_error_recovery(user_message: str) -> str:
    """Agent loop that handles tool errors gracefully."""
    messages = [{"role": "user", "content": user_message}]

    for iteration in range(10):
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=2048,
            tools=tools,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return ""

        tool_calls = [b for b in response.content if b.type == "tool_use"]
        if not tool_calls:
            return response.content[0].text

        messages.append({"role": "assistant", "content": response.content})

        # Execute tools with error handling
        tool_results = []
        for tool_call in tool_calls:
            success, result = safe_tool_execution(tool_call.name, tool_call.input)

            if success:
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": result
                })
            else:
                # Return error to model so it can adapt
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": result,  # Error message
                    "is_error": True
                })
                print(f"Tool {tool_call.name} failed: {result}")

        messages.append({"role": "user", "content": tool_results})

    return "Max iterations reached"
```

**Key pattern:** Return errors to the model, don't crash. The model can adapt:
- "Tool failed because network timeout" → Model retries
- "Tool failed because invalid input" → Model reformulates request
- "Tool failed because permission denied" → Model explains to user

---

## 4.3 Tool Validation Before Execution

```python
import re
from abc import ABC, abstractmethod

class ToolValidator(ABC):
    @abstractmethod
    def validate(self, tool_input: dict) -> tuple[bool, str]:
        """Returns: (is_valid, error_message)"""
        pass

class StockSymbolValidator(ToolValidator):
    def validate(self, tool_input: dict) -> tuple[bool, str]:
        symbol = tool_input.get("symbol", "").upper()
        if not symbol:
            return False, "Symbol is required"
        if not re.match(r"^[A-Z]{1,5}$", symbol):
            return False, f"Invalid symbol format: {symbol}"
        if symbol not in ["AAPL", "GOOGL", "MSFT"]:
            return False, f"Symbol {symbol} not supported"
        return True, ""

class MoneyTransferValidator(ToolValidator):
    def validate(self, tool_input: dict) -> tuple[bool, str]:
        amount = tool_input.get("amount")
        if amount is None:
            return False, "Amount is required"
        try:
            amount = float(amount)
        except (ValueError, TypeError):
            return False, f"Amount must be a number, got {type(amount).__name__}"
        if amount <= 0:
            return False, "Amount must be positive"
        if amount > 1_000_000:
            return False, "Amount exceeds maximum transfer limit ($1M)"

        recipient = tool_input.get("recipient")
        if not recipient:
            return False, "Recipient is required"

        return True, ""

# Tool registry with validators
TOOL_REGISTRY = {
    "get_stock_price": {
        "validator": StockSymbolValidator(),
        "executor": get_stock_price_impl
    },
    "transfer_funds": {
        "validator": MoneyTransferValidator(),
        "executor": transfer_funds_impl
    }
}

def execute_tool_with_validation(tool_name: str, tool_input: dict) -> str:
    """Validate input before execution."""
    if tool_name not in TOOL_REGISTRY:
        return f"Unknown tool: {tool_name}"

    config = TOOL_REGISTRY[tool_name]

    # Validate
    is_valid, error_msg = config["validator"].validate(tool_input)
    if not is_valid:
        return f"Validation error: {error_msg}"

    # Execute
    try:
        return config["executor"](tool_input)
    except Exception as e:
        return f"Execution error: {e}"
```

---

## 4.4 Tool Calling with Streaming

Stream token-by-token while the model is calling tools.

```python
def agent_loop_streaming(user_message: str):
    """Stream model response including tool calls as they arrive."""
    messages = [{"role": "user", "content": user_message}]

    # Streaming is powerful here — see tool calls in real-time
    with client.messages.stream(
        model="claude-opus-4-8",
        max_tokens=2048,
        tools=tools,
        messages=messages
    ) as stream:
        # Accumulate response content
        response_content = []
        current_tool_use_block = None

        for event in stream:
            # Various event types: content_block_start, content_block_delta, content_block_stop, message_stop

            if event.type == "content_block_start":
                if event.content_block.type == "tool_use":
                    current_tool_use_block = {
                        "id": event.content_block.id,
                        "name": event.content_block.name,
                        "input": ""
                    }
                    print(f"🔧 Tool call starting: {event.content_block.name}")

            elif event.type == "content_block_delta":
                if event.delta.type == "input_json_delta":
                    # Streaming the JSON input for the tool
                    current_tool_use_block["input"] += event.delta.partial_json
                    print(f"  Input chunk: {event.delta.partial_json}", end="")

                elif event.delta.type == "text_delta":
                    # Regular text streaming
                    print(event.delta.text, end="", flush=True)

            elif event.type == "content_block_stop":
                if current_tool_use_block:
                    response_content.append({
                        "type": "tool_use",
                        "id": current_tool_use_block["id"],
                        "name": current_tool_use_block["name"],
                        "input": json.loads(current_tool_use_block["input"])
                    })
                    current_tool_use_block = None

        # After streaming completes, have the full response
        print(f"\n✓ Stream complete. Executing {len([c for c in response_content if c.get('type') == 'tool_use'])} tools...")

        # Execute tools
        # ... same as before ...
```

**Advantage:** See tool invocations in real-time without waiting for full response. Great for long-running operations.

---

## 4.5 Conditional Tool Availability

Dynamically change available tools based on context.

```python
def get_available_tools(user_role: str, user_id: int) -> list[dict]:
    """
    Different users have access to different tools.
    Example: Admin can reset passwords, regular user cannot.
    """
    base_tools = [
        get_schema("get_user_profile"),
        get_schema("update_own_profile"),
    ]

    if user_role == "admin":
        base_tools.extend([
            get_schema("reset_user_password"),
            get_schema("suspend_user_account"),
            get_schema("view_system_logs"),
        ])

    if user_role in ["premium", "admin"]:
        base_tools.append(get_schema("get_advanced_analytics"))

    if user_id in get_beta_testers():
        base_tools.append(get_schema("use_beta_features"))

    return base_tools

def tool_aware_agent(user_id: int, user_role: str, user_message: str) -> str:
    """Agent with user-context-aware tool availability."""
    messages = [{"role": "user", "content": user_message}]
    available_tools = get_available_tools(user_role, user_id)

    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=2048,
        tools=available_tools,  # Dynamic tool list
        messages=messages,
        system=f"You are a helpful assistant for a {user_role} user."
    )

    # ... rest of agent loop ...
```

---

# 5. Production Reliability

## 5.1 Tool Call Logging & Auditing

```python
import logging
from datetime import datetime
import json

class ToolAuditLogger:
    def __init__(self, log_file: str = "tool_audit.jsonl"):
        self.log_file = log_file
        self.logger = logging.getLogger(__name__)

    def log_tool_call(
        self,
        user_id: str,
        tool_name: str,
        tool_input: dict,
        tool_output: str,
        execution_time_ms: float,
        success: bool
    ):
        """Log every tool call for audit trail."""
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "user_id": user_id,
            "tool_name": tool_name,
            "input": tool_input,
            "output": tool_output[:500],  # Truncate long outputs
            "execution_time_ms": execution_time_ms,
            "success": success
        }

        with open(self.log_file, "a") as f:
            f.write(json.dumps(entry) + "\n")

        # Also log to centralized system
        if success:
            self.logger.info(f"Tool {tool_name} succeeded in {execution_time_ms}ms")
        else:
            self.logger.error(f"Tool {tool_name} failed: {tool_output}")

    def get_user_tool_usage(self, user_id: str) -> dict:
        """Analyze tool usage for a user."""
        usage = {}

        with open(self.log_file, "r") as f:
            for line in f:
                entry = json.loads(line)
                if entry["user_id"] != user_id:
                    continue

                tool = entry["tool_name"]
                if tool not in usage:
                    usage[tool] = {"count": 0, "successes": 0, "avg_time_ms": 0}

                usage[tool]["count"] += 1
                if entry["success"]:
                    usage[tool]["successes"] += 1

        return usage

# Usage
audit_log = ToolAuditLogger()

def execute_tool_with_logging(
    user_id: str,
    tool_name: str,
    tool_input: dict
) -> str:
    import time
    start = time.time()

    success = False
    result = ""
    try:
        result = execute_tool(tool_name, tool_input)
        success = True
    except Exception as e:
        result = str(e)
    finally:
        elapsed_ms = (time.time() - start) * 1000
        audit_log.log_tool_call(
            user_id=user_id,
            tool_name=tool_name,
            tool_input=tool_input,
            tool_output=result,
            execution_time_ms=elapsed_ms,
            success=success
        )

    return result
```

---

## 5.2 Tool Call Rate Limiting

```python
from collections import defaultdict
from datetime import datetime, timedelta
import time

class RateLimiter:
    def __init__(
        self,
        max_calls_per_minute: int = 60,
        max_calls_per_hour: int = 1000
    ):
        self.max_calls_per_minute = max_calls_per_minute
        self.max_calls_per_hour = max_calls_per_hour
        self.call_times = defaultdict(list)  # user_id -> list of call timestamps

    def is_allowed(self, user_id: str) -> bool:
        """Check if user can make another tool call."""
        now = datetime.now()
        call_times = self.call_times[user_id]

        # Remove old entries
        one_hour_ago = now - timedelta(hours=1)
        one_minute_ago = now - timedelta(minutes=1)
        call_times[:] = [t for t in call_times if t > one_hour_ago]

        # Check minute limit
        recent_calls = [t for t in call_times if t > one_minute_ago]
        if len(recent_calls) >= self.max_calls_per_minute:
            return False

        # Check hour limit
        if len(call_times) >= self.max_calls_per_hour:
            return False

        return True

    def record_call(self, user_id: str):
        self.call_times[user_id].append(datetime.now())

    def get_reset_time(self, user_id: str) -> datetime:
        """When can the user make their next call?"""
        call_times = self.call_times[user_id]
        one_hour_ago = datetime.now() - timedelta(hours=1)
        old_calls = [t for t in call_times if t <= one_hour_ago]

        if len(old_calls) < self.max_calls_per_hour:
            # Hour limit not hit, check minute
            one_minute_ago = datetime.now() - timedelta(minutes=1)
            recent_calls = [t for t in call_times if t > one_minute_ago]
            if len(recent_calls) >= self.max_calls_per_minute:
                return recent_calls[0] + timedelta(minutes=1)

        # Hour limit hit
        return old_calls[-1] + timedelta(hours=1) if old_calls else datetime.now()


rate_limiter = RateLimiter(max_calls_per_minute=10, max_calls_per_hour=100)

def agent_loop_with_rate_limit(user_id: str, user_message: str) -> str:
    """Agent with rate limiting."""
    if not rate_limiter.is_allowed(user_id):
        reset_time = rate_limiter.get_reset_time(user_id)
        wait_seconds = (reset_time - datetime.now()).total_seconds()
        return f"Rate limit exceeded. Please retry in {wait_seconds:.0f} seconds."

    # ... rest of agent loop ...
    rate_limiter.record_call(user_id)
```

---

## 5.3 Tool Cost Tracking

```python
TOOL_COSTS = {
    "get_stock_price": {"api_cost": 0.01, "latency_ms": 200},
    "transfer_funds": {"api_cost": 0.05, "latency_ms": 500},
    "send_email": {"api_cost": 0.001, "latency_ms": 300},
}

class CostTracker:
    def __init__(self):
        self.costs_per_user = defaultdict(float)
        self.tool_usage = defaultdict(lambda: defaultdict(int))

    def record_tool_use(self, user_id: str, tool_name: str):
        cost = TOOL_COSTS.get(tool_name, {}).get("api_cost", 0.0)
        self.costs_per_user[user_id] += cost
        self.tool_usage[user_id][tool_name] += 1

    def get_user_cost(self, user_id: str) -> float:
        return self.costs_per_user[user_id]

    def get_user_budget_remaining(self, user_id: str, budget: float) -> float:
        return max(0, budget - self.get_user_cost(user_id))

    def should_allow_tool(
        self,
        user_id: str,
        tool_name: str,
        user_budget: float
    ) -> tuple[bool, str]:
        """Check if tool call would exceed budget."""
        cost = TOOL_COSTS.get(tool_name, {}).get("api_cost", 0.0)
        current_cost = self.get_user_cost(user_id)

        if current_cost + cost > user_budget:
            remaining = self.get_user_budget_remaining(user_id, user_budget)
            return False, f"Budget exceeded. ${remaining:.2f} remaining."

        return True, ""

cost_tracker = CostTracker()

def agent_with_cost_control(
    user_id: str,
    user_message: str,
    user_budget: float = 10.0
) -> str:
    """Agent that respects user budget."""
    messages = [{"role": "user", "content": user_message}]

    for iteration in range(10):
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=1024,
            tools=tools,
            messages=messages,
            system=f"You have a budget of ${user_budget:.2f}. Use tools wisely."
        )

        tool_calls = [b for b in response.content if b.type == "tool_use"]

        if not tool_calls:
            for b in response.content:
                if hasattr(b, "text"):
                    return b.text

        # Check budget before executing any tools
        for tool_call in tool_calls:
            allowed, msg = cost_tracker.should_allow_tool(
                user_id,
                tool_call.name,
                user_budget
            )
            if not allowed:
                return f"Cannot proceed: {msg}"

        messages.append({"role": "assistant", "content": response.content})

        # Execute and track cost
        tool_results = []
        for tool_call in tool_calls:
            result = execute_tool(tool_call.name, tool_call.input)
            cost_tracker.record_tool_use(user_id, tool_call.name)

            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tool_call.id,
                "content": result
            })

        messages.append({"role": "user", "content": tool_results})

    return "Completed"
```

---

# 6. Interview Q&A

## Q1: Tool Selection Mistakes — What Went Wrong?

**Scenario:** Your agent called `update_user_email` with `email="support@company.com"` and the tool returned an error: "Email already in use." But the user said "Change my email to support@company.com" — how do you handle this?

**Strong Answer:**
> The model correctly identified that the tool should be called, but the input validation caught a legitimate error: the target email is already in use. I would:
>
> 1. **Return the error to the model.** Don't hide errors — send back `"Email already in use. This email is registered to another account."` as a `tool_result` with `is_error=true`.
>
> 2. **Let the model adapt.** The model will reason: "The user asked for this email, but it's taken. I should explain this to the user and ask for an alternative."
>
> 3. **Don't retry automatically.** This isn't a transient failure; retrying the same input won't help.
>
> This is actually the agent working correctly — catching an invalid operation before it happens. The flow is: tool validation → error → model adapts → asks user for clarification.

---

## Q2: When Would You NOT Use Tool Calling?

**Strong Answer:**
> Tool use adds latency (model→tool→model roundtrip) and complexity (error handling, validation). Skip it when:
>
> 1. **The answer is in your system prompt or context.** If you already injected all needed data, don't call a tool to fetch it again.
>
> 2. **The model can reason to the answer directly.** "What is 2+2?" doesn't need a calculator tool; the model predicts "4" faster than calling a tool.
>
> 3. **The operation is irreversible and risky.** For high-stakes writes (delete accounts, financial transfers), use explicit user confirmation UI, not tool calling. Don't trust the model to make these decisions.
>
> 4. **Latency is critical.** If you need sub-100ms responses, tool roundtrips kill performance. Cache or precompute instead.
>
> Rule of thumb: Use tools for **information retrieval, calculations the model can't do, or state mutations the user explicitly requested.** Don't use tools for reasoning that the model can do in its head.

---

## Q3: How Do You Debug Tool Calling Issues?

**Strong Answer:**
> Three levels of debugging:
>
> **Level 1: Tool Call Extraction**
> - Is the model outputting tool calls at all? Check: is `stop_reason == "tool_use"`?
> - Are tool names spelled correctly? Check model output against tool definitions.
> - Inspect the actual tool_input dict — is it valid JSON?
>
> **Level 2: Tool Execution**
> - Does the tool return an error or success?
> - Log every tool call: input, output, latency. Audit logs are essential.
> - Check: did the tool actually do what we expected? (E.g., did the database write succeed?)
>
> **Level 3: Model Response**
> - After feeding the tool result back, does the model misinterpret it?
> - Is the model output format correct?
> - Is the model hallucinating or acknowledging the tool result?
>
> **Red flags:**
> - Model calls the same tool repeatedly with the same input (infinite loop).
> - Model calls tools that don't exist (check schema names).
> - Model calls tools with missing required fields (schema definition is unclear).
> - Tool succeeds but model says it failed (model misread the result).
>
> Always log: user_id, tool_name, input, output, latency, success/fail. Audit logs catch 90% of issues.

---

## Q4: Tool Calling with External APIs — How Do You Handle Timeouts?

**Strong Answer:**
> Tool calls to external APIs can hang, timeout, or fail unpredictably. My strategy:
>
> ```python
> def call_external_api(endpoint: str, params: dict, timeout_sec: int = 5) -> str:
>     try:
>         response = requests.get(
>             endpoint,
>             params=params,
>             timeout=timeout_sec  # Always set timeout
>         )
>         response.raise_for_status()
>         return response.json()
>     except requests.Timeout:
>         return f"API timeout after {timeout_sec}s. Try again later."
>     except requests.ConnectionError as e:
>         return f"Cannot reach API: {e}. Using cached data if available."
>     except requests.HTTPError as e:
>         return f"API returned error: {e.response.status_code}"
> ```
>
> Then in the agent loop, I return the error message to the model:
> ```python
> tool_result = {
>     "type": "tool_result",
>     "tool_use_id": tool_call.id,
>     "content": error_message,  # "API timeout..."
>     "is_error": True
> }
> ```
>
> The model reads "API timeout" and can decide: retry, use fallback data, explain to user.
>
> **Key principle:** Errors are first-class citizens. Return them as tool results, let the model adapt.

---

## Q5: Parallel Tool Calls vs Sequential — When to Use Each?

**Strong Answer:**
> **Parallel tool calls:**
> - Model calls multiple tools in one response
> - You execute all of them at once (no dependencies)
> - Faster (N tools in ~same time as 1 tool)
> - Use when tools are independent (get_stock_price + get_company_info)
>
> **Sequential tool calls:**
> - First tool call output feeds into the second
> - Tool 2 depends on result of Tool 1
> - Use when there are explicit dependencies
>
> **Example:**
> ```
> Sequential: get_user_id(email) → get_user_profile(user_id)
> (Can't get profile without ID from first call)
>
> Parallel: get_stock_price(AAPL) + get_stock_price(GOOGL) + get_news()
> (All independent, run together)
> ```
>
> In Claude's tool-use system, you don't specify parallel vs. sequential — the model decides. If it calls multiple tools in one response, they're independent. If they're sequential, it waits for one result before calling the next.
>
> **Optimization:** If the model keeps calling tools sequentially when they could be parallel, explicitly tell it: "You can call multiple tools at once if they don't depend on each other."

---

## Q6: Security in Tool Calling — How Do You Prevent Abuse?

**Strong Answer:**
> Three layers:
>
> **Layer 1: Input Validation**
> - Every tool input is validated against schema before execution.
> - Reject: null values where not allowed, strings > max length, numbers out of range.
> - Whitelist over blacklist — explicitly allow valid values, reject everything else.
>
> **Layer 2: Authorization**
> - Not every user can call every tool.
> - Check user role/permissions before executing: `if tool == "reset_password" and user_role != "admin": return "Permission denied"`
> - Log all permission failures.
>
> **Layer 3: Rate Limiting & Budgeting**
> - Limit tool calls per user per minute/hour.
> - Track costs and enforce budgets (prevent API bill runaway).
> - Require explicit confirmation for dangerous operations (delete, transfer, reset password).
>
> **Example:**
> ```python
> # Don't let the model call sensitive tools without user consent
> if tool_name in ["delete_account", "transfer_funds"]:
>     return f"This operation requires user confirmation. Ask the user: '{tool_name.replace('_', ' ')}?'"
> ```
>
> **Golden rule:** The model is a reasoning engine, not a security boundary. You (the system) enforce permissions.

---

## Q7: When Tool Arguments Are Ambiguous or Missing?

**Scenario:** User says "Send me a notification" but doesn't specify what kind (email, SMS, push). The tool requires `notification_type`. How do you handle this?

**Strong Answer:**
> Two approaches:
>
> **Option 1: Default + Ask**
> ```python
> if "notification_type" not in tool_input:
>     return "Notification type not specified. I'll use 'email'. Alternatives: email, sms, push."
> ```
> Return this to the model as a tool result, and it will inform the user.
>
> **Option 2: Prompt the Model to Ask**
> ```python
> if "notification_type" not in tool_input:
>     return "Cannot send notification: notification_type is required. 
>             Ask the user: 'How should I send the notification? (email/sms/push)'"
> ```
> The model then asks the user before calling the tool again.
>
> **Preferred: Option 2.** Don't make assumptions; let the model clarify with the user. This respects user intent and catches errors early.
>
> **Prevention:** In the tool schema, mark required fields clearly. In the system prompt, instruct the model: "If a required tool argument is missing or ambiguous, ask the user before calling the tool."

---

## Q8: Streaming Tool Calls — When Does It Help?

**Strong Answer:**
> Streaming tool calls is useful when:
> 1. **Long tool execution.** You're calling an API that takes 5+ seconds. While you wait, stream reasoning text to the user ("Let me fetch that data...").
> 2. **Token transparency.** Users see tool calls as they happen (good UX for debugging/transparency).
> 3. **Multi-step workflows.** Tool 1 takes 2s, tool 2 takes 3s. Stream progress between them.
>
> **When NOT to stream:**
> - Tool calls are fast (<500ms).
> - You need to make decisions based on all tool results before responding.
> - Low-latency is critical (streaming adds overhead).
>
> **Implementation:**
> ```python
> with client.messages.stream(..., tools=tools) as stream:
>     for event in stream:
>         if event.type == "content_block_start" and event.content_block.type == "tool_use":
>             print(f"Calling {event.content_block.name}...")
> ```
>
> This shows the user "Calling [tool]" before waiting for the full response.

---

## Q9: Tool Calling in RAG — How Do You Combine Them?

**Strong Answer:**
> RAG (Retrieval-Augmented Generation) and tool calling often complement each other:
>
> **Pattern 1: Tool Calls to Retrieve**
> The tool IS the retriever:
> ```python
> {
>     "name": "search_knowledge_base",
>     "description": "Search internal documentation for answers",
>     "input_schema": {...}
> }
> # Model calls this, you execute vector search, return results
> ```
>
> **Pattern 2: RAG + Tools for Actions**
> RAG provides context; tools execute actions:
> - RAG retrieves product info from docs → model understands
> - Tool calls place_order(product_id) → actually creates order
>
> **Pattern 3: Tool to Validate RAG**
> Tool verifies information from retrieval:
> - RAG retrieves: "Product X is in stock"
> - Tool calls verify_stock(product_id) → "Actually, stock is 0"
> - Model corrects itself
>
> **Combined flow:**
> ```
> User: "Can I buy product X?"
> 1. RAG retrieves product info
> 2. Model calls verify_stock(X)
> 3. Tool returns inventory data
> 4. Model combines RAG + tool data → answer
> ```
>
> **Key principle:** RAG = knowledge retrieval (what is X?). Tools = state queries/mutations (does X exist? change Y). Use both.

---

## Q10: Common Tool Calling Failures — Root Causes?

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| Model never calls tools | Tool definitions are missing/hidden, or model thinks it doesn't need them | Add tools explicitly; include in system prompt: "You have access to these tools. Use them when needed." |
| Model calls wrong tool repeatedly | Schema is confusing; two tools overlap | Rename tools for clarity; one tool per responsibility |
| Tool execution succeeds but model says it failed | Tool returned error string instead of success; model misread | Return clear success/error indicators: `{"status": "success", "data": ...}` or `{"error": "..."}`  |
| Tool input is always invalid JSON | Streaming tool inputs are malformed | Validate/parse tool input carefully; retry with error message |
| Parallel tool calls fail silently | One tool errors, execution stops | Execute all tools, return individual results (don't stop on first error) |
| Tool gets called 100x in a loop | Max iteration limit is too high, or tool keeps failing | Set max_iterations to 10-15; return clear errors so model doesn't retry forever |

---

*End of Tool Use & Function Calling Master Guide*

---

**Next in Series:**
- File 1: Core Foundations (Prompt Engineering, Tokens, Embeddings)
- **File 2: This File (Tool Use & Function Calling)**
- File 3: Agentic Systems & ReAct
- File 4: Production Deployment & Evaluation

