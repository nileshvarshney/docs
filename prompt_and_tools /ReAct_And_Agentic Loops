# 🤔➡️🔧 ReAct + Agentic Loops: Complete Master Guide
## Deep Dive into Reasoning + Acting and Autonomous Agent Architecture

> **Level:** Senior-level understanding (suitable for interviews and production systems)  
> **Scope:** This is THE definitive guide to agentic systems — everything needed, nothing extra  
> **Format:** Self-contained. Mental models → Implementation → Patterns → Troubleshooting

---

## 📋 Table of Contents

### Part 1: Mental Models & Foundations
- [1.1: What Is ReAct?](#11-what-is-react)
- [1.2: The Core Insight (Why ReAct Works)](#12-the-core-insight-why-react-works)
- [1.3: ReAct vs Traditional Approaches](#13-react-vs-traditional-approaches)
- [1.4: Agentic Loops Defined](#14-agentic-loops-defined)

### Part 2: ReAct Pattern Deep Dive
- [2.1: The THINK → ACT → OBSERVE Cycle](#21-the-think--act--observe-cycle)
- [2.2: System Prompts for ReAct](#22-system-prompts-for-react)
- [2.3: Why Explicit Reasoning Matters](#23-why-explicit-reasoning-matters)
- [2.4: ReAct With Tool Calling](#24-react-with-tool-calling)

### Part 3: Agentic Loop Architecture
- [3.1: The Agent Loop Flow](#31-the-agent-loop-flow)
- [3.2: State Management](#32-state-management)
- [3.3: Iteration Limits & Termination](#33-iteration-limits--termination)
- [3.4: Error Recovery in Loops](#34-error-recovery-in-loops)

### Part 4: Implementation Patterns
- [4.1: Basic ReAct Agent](#41-basic-react-agent)
- [4.2: Stateful Agent with Tracking](#42-stateful-agent-with-tracking)
- [4.3: Advanced Patterns](#43-advanced-patterns)
- [4.4: Common Implementation Mistakes](#44-common-implementation-mistakes)

### Part 5: Production Considerations
- [5.1: Debugging Agents](#51-debugging-agents)
- [5.2: Monitoring & Observability](#52-monitoring--observability)
- [5.3: Cost Management](#53-cost-management)
- [5.4: Reliability & Resilience](#54-reliability--resilience)

### Part 6: Interview Q&A
- [6.1: Core Concept Questions](#61-core-concept-questions)
- [6.2: Implementation Questions](#62-implementation-questions)
- [6.3: Production Questions](#63-production-questions)

---

# Part 1: Mental Models & Foundations

## 1.1 What Is ReAct?

### The Simple Definition

**ReAct = Reasoning + Acting**

It's a pattern where the model **interleaves thinking with tool calls** instead of planning everything upfront.

### Let's Compare Two Approaches

#### ❌ Without ReAct (Plan-Then-Execute)

```
User: "Find me a cheap flight from NYC to LA tomorrow, and tell me if it's raining there."

Model's internal process:
"I need to: (1) search flights, (2) get weather. Let me get both."

Model output:
"I will search for flights and check weather."

Reality check:
❌ Model doesn't know if flights exist
❌ Model can't actually check weather
❌ Often hallucinates information
```

#### ✅ With ReAct (Think-Act-Think-Act...)

```
User: "Find me a cheap flight from NYC to LA tomorrow, and tell me if it's raining there."

Model's process:
[THINK] I need to find flights and check weather. Let me start with flights.
[ACT] Call search_flights(from="NYC", to="LA", date="tomorrow")
[OBSERVE] Found flights: Delta $120, United $150, Southwest $110
[THINK] Southwest is cheapest at $110. Now I need the weather.
[ACT] Call get_weather(location="LA", date="tomorrow")
[OBSERVE] Weather: 75°F, sunny, 0% rain
[THINK] I have all the info now. Let me provide a complete answer.
[ANSWER] Southwest has the cheapest flight at $110. LA will be sunny tomorrow.

Reality check:
✅ Reasoning is explicit and auditable
✅ Tool calls are grounded in actual data
✅ Model can adapt if tools fail or return unexpected data
```

### The Key Difference

| Aspect | Without ReAct | With ReAct |
|--------|---|---|
| **Reasoning** | Hidden in model weights | Explicit in output |
| **Tool selection** | Decided upfront | Adaptive (based on observations) |
| **Error recovery** | Hard to debug | Clear where it failed |
| **Token efficiency** | Maybe fewer tokens initially | More tokens, but higher quality |
| **Auditability** | Black box | Transparent reasoning chain |

---

## 1.2 The Core Insight (Why ReAct Works)

### Why Does Adding Explicit Reasoning Help?

This seems counterintuitive. Doesn't adding intermediate steps cost more tokens and latency?

**Yes — but the quality gain is worth it.**

### The Mechanism

Think of the model as a **next-token predictor**. Every token the model generates becomes context for the next token.

```
Without intermediate thinking:
"What's the weather in LA?" 
→ [Model predicts] "Sunny"  ❌ (just a guess)

With intermediate thinking:
"What's the weather in LA?"
→ [Model predicts] "Let me check the weather API..."
→ [Tool returns] "75°F, sunny"
→ [Model continues predicting] "The weather in LA is sunny, 75°F"  ✅ (grounded in data)
```

### Two Powerful Effects

**1. Grounding:** When the model sees actual tool outputs, it uses those for reasoning instead of making up answers.

**2. Self-correction:** If a tool call fails, the model sees the error and tries a different approach.

```
Example with self-correction:
[THINK] Let me search for flights
[ACT] Call search_flights(NYC, LA, tomorrow)
[OBSERVE] Error: Tomorrow's flights not available yet
[THINK] The API doesn't have tomorrow's flights. Let me try in 2 days.
[ACT] Call search_flights(NYC, LA, in_2_days)
[OBSERVE] Found flights: ...
```

### Why This Matters in Practice

**Without ReAct:** Model hallucinates → you get wrong answers → customer is unhappy

**With ReAct:** Model tries → sees error → adapts → gives correct answer → customer is happy

---

## 1.3 ReAct vs Traditional Approaches

### Five Classic Patterns (and How ReAct Fits)

#### Pattern 1: Simple Prompting (Baseline)

```
System: "You are a helpful assistant."
User: "What's the weather tomorrow?"
Model: [Makes up an answer]  ❌
```

**Problem:** No grounding, pure hallucination.

#### Pattern 2: Few-Shot Learning (Improvement)

```
System: "Answer questions accurately.
Example: Q: Weather tomorrow? A: I'd need to check a weather API."
User: "What's the weather tomorrow?"
Model: "I'd need to check..." ✅ (but doesn't actually check)
```

**Problem:** Model knows what it *should* do, but doesn't do it.

#### Pattern 3: Tool Calling (Progress)

```
System: "You have access to: weather_api(location)"
User: "What's the weather tomorrow?"
Model: [Calls weather_api("current_location")]
Model: "It's sunny and 72°F"  ✅
```

**Problem:** Model doesn't explain reasoning. If it calls the wrong tool, you don't see why.

#### Pattern 4: ReAct (The Full Picture)

```
System: "You reason before acting. Format: [THINK] ... [ACT] ... [OBSERVE] ..."
User: "What's the weather tomorrow?"
Model: [THINK] I need weather for tomorrow. Let me call the weather API.
        [ACT] Call weather_api("current_location", "tomorrow")
        [OBSERVE] {"temp": 72, "condition": "sunny"}
        The weather tomorrow will be sunny, 72°F  ✅✅
```

**Benefit:** Reasoning is explicit, auditable, and self-correcting.

#### Pattern 5: Chain-of-Thought (Different Focus)

```
System: "Think step by step before answering."
User: "If A=2, B=3, what is A²+B³?"
Model: [THINK] A²=2²=4. B³=3³=27. Total = 4+27=31
        Answer: 31  ✅
```

**Difference from ReAct:** CoT is for math/logic. ReAct is for tool-based tasks.

### Comparison Table

| Pattern | Use Case | Hallucination Risk | Auditability | Overhead |
|---------|----------|---|---|---|
| Simple prompting | Trivia | ⚠️  High | 🔴 Low | Minimal |
| Few-shot | Instruction following | ⚠️  Medium | 🟡 Medium | Token cost |
| Tool calling | API integration | 🟢 Low | 🟡 Medium | Latency cost |
| **ReAct** | **Complex reasoning** | **🟢 Very low** | **🟢 High** | **Medium** |
| Chain-of-Thought | Math/logic | 🟢 Low | 🟢 High | Token cost |

---

## 1.4 Agentic Loops Defined

### What Is an Agent Loop?

An **agentic loop** is the repetitive structure where:
1. Model reasons about current state
2. Model decides what tool to call (or stop)
3. You execute the tool
4. Loop back to step 1 with new information

### The Diagram

```
        ┌─────────────────────────┐
        │ Start: User Query       │
        │ + Conversation History  │
        └────────────┬────────────┘
                     ↓
        ┌─────────────────────────┐
        │ Model: Reason           │
        │ (What should I do next?)│
        └────────────┬────────────┘
                     ↓
        ┌─────────────────────────┐
        │ Model Output:           │
        │ - Thinking text         │
        │ - Tool call (or END)    │
        └────────────┬────────────┘
                     ↓
            ┌───────┴───────┐
            ↓               ↓
      [Tool Call]      [END_TURN]
            ↓               ↓
        ┌────────┐   ┌──────────┐
        │Execute │   │Return    │
        │ Tool   │   │Answer    │
        └────┬───┘   └──────────┘
             ↓
    ┌────────────────┐
    │Tool Result     │
    │(observation)   │
    └────────┬───────┘
             ↓
      [Loop continues]
```

### Key Terms

| Term | Definition | Example |
|------|-----------|---------|
| **Iteration** | One cycle of the loop | Model reasons → calls tool → gets result → next iteration |
| **State** | Information carried between iterations | Conversation history, tool results, metadata |
| **Stop reason** | Why the loop halted | "end_turn" (model is done), "max_iterations" (safety limit), "error" |
| **Observation** | Tool result fed back to model | JSON response, error message, empty result |

### Why Loops Are Necessary

Some tasks require sequential steps:

```
Task: "Plan a trip from NYC to LA, including flights and hotels."

Sequential dependencies:
1. Search flights NYC→LA (must complete)
2. Get hotel options in LA (depends on arrival date from step 1)
3. Calculate total cost (depends on both previous steps)

Can't do all at once — must loop sequentially.
```

---

# Part 2: ReAct Pattern Deep Dive

## 2.1 The THINK → ACT → OBSERVE Cycle

### The Three Phases Explained

#### Phase 1: THINK (Model Reasoning)

Model analyzes the problem and decides what to do next.

```
Example THINK:
"I need to find information about Paris' population.
The user asked for current data, so I should search for recent statistics
rather than relying on my training data (which might be outdated).
Let me start with a web search."

Characteristics:
✅ Explicit
✅ Shows intent
✅ Explains next step
✅ Can be wrong (that's okay — we'll observe and correct)
```

**Mental model:** THINK is the model saying "here's my plan and why."

#### Phase 2: ACT (Tool Execution)

Model calls a tool. You execute it and return the result.

```
Example ACT:
Model: "I'll call search_web("Paris population 2024")"

What happens:
1. Model generates tool call (JSON/function format)
2. You parse it and validate
3. You execute the actual tool
4. You capture the result

Key point: Model doesn't execute the tool. You do.
This is crucial for security and reliability.
```

**Mental model:** ACT is the model "reaching out" to the external world.

#### Phase 3: OBSERVE (Result Integration)

Model receives and processes the tool result.

```
Example OBSERVE:
Tool result: {
  "sources": [
    {"title": "Paris population 2024", "data": "Population: 2.1M"}
  ]
}

Model reads this and updates its understanding.
Next THINK step will incorporate this new information.
```

**Mental model:** OBSERVE is the model "seeing" the external world's response.

### Why This Cycle Repeats

```
Iteration 1:
THINK: "I need population data"
ACT: search_web("Paris population")
OBSERVE: Got 2.1M

Iteration 2:
THINK: "I have population. User also asked about economy. Let me search."
ACT: search_web("Paris economy 2024")
OBSERVE: Got economic data

Iteration 3:
THINK: "I have enough info. Let me synthesize answer."
ACT: [No tool call — generate final answer]
OBSERVE: [End turn]
```

---

## 2.2 System Prompts for ReAct

### Core Elements of a ReAct System Prompt

```python
REACT_SYSTEM_PROMPT = """You are an expert problem-solver using the ReAct (Reasoning + Acting) pattern.

## Your Process

1. **THINK**: Analyze the current situation. What information do I have? 
   What's still missing? What's my next step?

2. **ACT**: Call a tool if you need information. Or provide your final answer
   if you have enough information.

3. **OBSERVE**: Carefully read the tool result. Does it answer your question?
   Does it raise new questions? Update your understanding.

4. **REPEAT**: If you don't have a complete answer, go back to THINK.

## Format

<thinking>
[Your reasoning about what to do next]
</thinking>

<action>
[Tool call or final answer]
</action>

## Important Rules

- Always THINK before acting. Don't jump to conclusions.
- OBSERVE means carefully reading tool results, not skimming them.
- If a tool returns an error, that's an observation. Adapt your approach.
- You have access to these tools: [list tools]
- You have a maximum of 10 iterations. After that, synthesize the best answer 
  you can from available information.

## Examples

[Include 1-2 full examples of good ReAct chains]
"""
```

### What Makes a Good ReAct Prompt?

| Element | Good | Bad |
|---------|------|-----|
| **Explicitness** | "Always THINK before acting" | "Use tools when needed" |
| **Format** | Uses XML tags or clear delimiters | Implicit formatting |
| **Rules** | Clear iteration limits | No stopping criteria |
| **Examples** | Shows full THINK-ACT-OBSERVE-THINK chain | Shows only successful case |
| **Tool list** | Describes each tool clearly | Just lists tool names |

---

## 2.3 Why Explicit Reasoning Matters

### Reason #1: Debugging

When something goes wrong, you can see exactly where:

```
❌ Without ReAct:
Model: "The population of Paris is 3M."
User: "That's wrong. It's 2.1M."
You: "I don't know why it was wrong. No visibility."

✅ With ReAct:
[THINK] "I need current population..."
[ACT] search_web("Paris population")
[OBSERVE] Results show 2.1M
[ANSWER] "Paris population is 2.1M"
User: "Correct!"
You: "I can see it searched and found the right answer."

If it was wrong:
[THINK] "I need current population..."
[ACT] search_web("Paris population 2024")  ← Maybe this query is bad
[OBSERVE] Got 3M (from old cached results)
[THINK] "Hmm, 3M seems high. Let me verify with another source"
↑ You could see the model's self-doubt and fix the search query
```

### Reason #2: Self-Correction

With explicit reasoning visible, the model can correct itself:

```
[THINK] "Let me search for flights from NYC to LA"
[ACT] call search_flights(from="NYC", to="LA", date="tomorrow")
[OBSERVE] Error: "Tomorrow's date is in the past. Please provide a future date."
[THINK] "Oh! The API rejected my query. Today must be later than I thought.
         Let me try for the day after tomorrow instead."
[ACT] call search_flights(from="NYC", to="LA", date="in_2_days")
[OBSERVE] Success: [list of flights]
```

**Without explicit reasoning,** the model might just say "I can't find flights" and give up.  
**With ReAct,** it adapts and retries intelligently.

### Reason #3: Compliance & Auditing

In regulated industries (finance, healthcare), you need to show your work:

```
Example: Medical diagnosis system
[THINK] "Patient reports headache and fever. These could indicate flu or COVID.
         I should check their symptoms, medical history, and recent exposure."
[ACT] call get_patient_history(patient_id)
[OBSERVE] History shows: vaccinated, no recent travel
[THINK] "Vaccination history is relevant. Let me check current symptoms in detail."
[ACT] call analyze_symptoms(symptoms=["headache", "fever"])
[OBSERVE] Results suggest: 70% flu, 20% COVID, 10% other

[ANSWER] "Most likely flu given vaccination status. Recommend: [treatments]
          This recommendation is based on: [chain of reasoning shown above]"

Auditor can verify: 
✅ Reasoning was sound
✅ Tools were used correctly
✅ Conclusion matches the evidence
```

### Reason #4: Trust & Transparency

Users trust systems they can understand:

```
❌ Opaque system:
User: "Why did you recommend Product X?"
System: "Based on my analysis."
User: "That doesn't help. I don't trust it."

✅ ReAct system:
User: "Why did you recommend Product X?"
System: [Shows full reasoning chain]
        "You said you value: cost-effectiveness, durability, and style.
         I searched for products matching these criteria.
         Product X scored: 9/10 cost, 8/10 durability, 9/10 style.
         Products Y and Z scored lower on your priorities.
         Here's my reasoning: [detailed chain]"
User: "That makes sense! I can see exactly why you chose it."
```

---

## 2.4 ReAct With Tool Calling

### How They Fit Together

**ReAct** is the pattern (THINK-ACT-OBSERVE).  
**Tool calling** is the mechanism (how the model requests tool execution).

```
ReAct flow + Tool calling:

[THINK] Model outputs reasoning text
[ACT] Model outputs tool_use block with:
      - tool_name: "search_web"
      - arguments: {"query": "..."}
[OBSERVE] You execute tool, return result as tool_result message
[Repeat]
```

### Practical Implementation

```python
from anthropic import Anthropic

client = Anthropic()

TOOLS = [
    {
        "name": "search_web",
        "description": "Search the web for current information",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query"
                }
            },
            "required": ["query"]
        }
    }
]

SYSTEM_PROMPT = """You are a researcher using ReAct (Reasoning + Acting).

Format your responses:
<thinking>
[Your reasoning about what to do next]
</thinking>

Then either:
A) Call a tool if you need information
B) Provide your final answer if you have enough information

Important: Always THINK before deciding to act or answer."""

def react_agent(user_query: str) -> str:
    """A simple ReAct agent."""
    messages = [{"role": "user", "content": user_query}]
    
    for iteration in range(10):  # Max 10 iterations
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=2048,
            system=SYSTEM_PROMPT,
            tools=TOOLS,
            messages=messages
        )
        
        # Check if done
        if response.stop_reason == "end_turn":
            # Model finished — extract final answer
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return ""
        
        # Extract thinking and tool calls
        thinking = ""
        tool_calls = []
        
        for block in response.content:
            if hasattr(block, "text"):
                thinking = block.text
            if block.type == "tool_use":
                tool_calls.append(block)
        
        if not tool_calls:
            # No tools called — return final answer
            return thinking
        
        # Add assistant response to history
        messages.append({"role": "assistant", "content": response.content})
        
        # Execute tools (in this example, we'll mock them)
        tool_results = []
        for tool_call in tool_calls:
            # Mock tool execution
            if tool_call.name == "search_web":
                result = f"Search results for: {tool_call.input['query']}"
            else:
                result = "Tool not found"
            
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tool_call.id,
                "content": result
            })
        
        # Feed results back
        messages.append({"role": "user", "content": tool_results})
    
    return "Max iterations reached"

# Example usage
answer = react_agent("Who won the 2024 US Presidential election?")
print(answer)
```

---

# Part 3: Agentic Loop Architecture

## 3.1 The Agent Loop Flow

### Step-by-Step Breakdown

Let me walk you through what happens at each step:

```
Iteration 1: Initial Call
├─ Input: User message
├─ Model reasoning: Decides first action
├─ Output: Reasoning text + tool call OR final answer
└─ Decision: Continue loop or stop?

Iteration 2: Tool Execution
├─ Input: Previous messages + tool result
├─ Model reasoning: Incorporates new data
├─ Output: Reasoning text + another tool call OR final answer
└─ Decision: Continue loop or stop?

...repeat...

Iteration N: Final Answer
├─ Input: Accumulated messages and observations
├─ Model reasoning: Synthesizes all information
├─ Output: No tool call, just final answer
└─ Decision: STOP
```

### Key Point: Message History

The message history is like the agent's memory. It accumulates:

```
messages = [
    {"role": "user", "content": "Original question"},
    {"role": "assistant", "content": "Thinking... [tool_use block]"},
    {"role": "user", "content": "tool_result from execution"},
    {"role": "assistant", "content": "Thinking more... [tool_use block]"},
    {"role": "user", "content": "another tool_result"},
    ...
]
```

Each message is context for the next model call. This is how the agent "remembers" what it discovered.

---

## 3.2 State Management

### What State Should the Agent Track?

For basic agents, you need:

```python
class AgentState:
    messages: list        # Conversation history
    iteration: int        # Current loop count
    max_iterations: int   # Safety limit
    tools_called: list    # Record of what was called
    start_time: datetime  # For timeout detection
    results: dict         # Final outputs
```

For more sophisticated agents:

```python
class AdvancedAgentState:
    messages: list        # Conversation history
    iteration: int        # Current iteration
    max_iterations: int   # Safety limit
    
    # Detailed tracking
    tool_calls: list[ToolCallRecord]  # [tool_name, input, output, latency]
    reasoning_chain: str   # Full THINK-ACT-OBSERVE narrative
    errors_encountered: list[str]  # Errors that occurred
    
    # Context
    user_id: str
    session_id: str
    started_at: datetime
    metadata: dict         # Custom data
    
    # Outputs
    final_answer: str
    confidence: float      # How confident in the answer?
```

### Practical Example

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class ToolCallRecord:
    """Track a single tool invocation."""
    iteration: int
    tool_name: str
    input: dict
    output: str
    latency_ms: float
    success: bool
    timestamp: datetime

class StatefulAgent:
    def __init__(self, max_iterations: int = 10):
        self.max_iterations = max_iterations
        self.state = {
            "messages": [],
            "iteration": 0,
            "tool_calls": [],
            "reasoning_chain": "",
            "final_answer": "",
            "errors": []
        }
    
    def run(self, user_query: str):
        """Execute the agent loop with full state tracking."""
        self.state["messages"] = [{"role": "user", "content": user_query}]
        self.state["started_at"] = datetime.now()
        
        for iteration in range(self.max_iterations):
            self.state["iteration"] = iteration
            
            # Call model
            response = client.messages.create(
                model="claude-opus-4-8",
                max_tokens=2048,
                messages=self.state["messages"]
            )
            
            # Extract reasoning
            reasoning_text = ""
            for block in response.content:
                if hasattr(block, "text"):
                    reasoning_text += block.text
            self.state["reasoning_chain"] += f"\n[Iteration {iteration}]\n{reasoning_text}"
            
            # Check if done
            if response.stop_reason == "end_turn":
                self.state["final_answer"] = reasoning_text
                self.state["messages"].append({
                    "role": "assistant",
                    "content": response.content
                })
                break
            
            # Process tool calls
            tool_calls = [b for b in response.content if b.type == "tool_use"]
            
            if not tool_calls:
                self.state["final_answer"] = reasoning_text
                break
            
            self.state["messages"].append({"role": "assistant", "content": response.content})
            
            # Execute tools and track
            tool_results = []
            for tool_call in tool_calls:
                start = datetime.now()
                try:
                    result = self._execute_tool(tool_call.name, tool_call.input)
                    success = True
                except Exception as e:
                    result = str(e)
                    success = False
                    self.state["errors"].append(f"Tool {tool_call.name} failed: {e}")
                
                latency = (datetime.now() - start).total_seconds() * 1000
                
                # Record this tool call
                record = ToolCallRecord(
                    iteration=iteration,
                    tool_name=tool_call.name,
                    input=tool_call.input,
                    output=result,
                    latency_ms=latency,
                    success=success,
                    timestamp=datetime.now()
                )
                self.state["tool_calls"].append(record)
                
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": result
                })
            
            self.state["messages"].append({"role": "user", "content": tool_results})
        
        return self.state

    def _execute_tool(self, tool_name: str, tool_input: dict) -> str:
        """Execute a tool (implement based on your tools)."""
        if tool_name == "search":
            return f"Results for: {tool_input['query']}"
        return "Unknown tool"
```

### What You Can Do With State

```python
# Analytics
def get_loop_metrics(state):
    return {
        "total_iterations": state["iteration"],
        "tools_called": len(state["tool_calls"]),
        "successful_tools": sum(1 for t in state["tool_calls"] if t.success),
        "total_latency_ms": sum(t.latency_ms for t in state["tool_calls"]),
        "errors": len(state["errors"])
    }

# Debugging
def debug_agent(state):
    print("=== REASONING CHAIN ===")
    print(state["reasoning_chain"])
    print("\n=== TOOL CALLS ===")
    for call in state["tool_calls"]:
        print(f"{call.tool_name}: {call.input} → {call.output[:100]}")
    if state["errors"]:
        print("\n=== ERRORS ===")
        for err in state["errors"]:
            print(f"- {err}")

# Cost tracking
def calculate_cost(state):
    input_tokens = sum(len(msg.get("content", "").split()) * 1.3 for msg in state["messages"])
    # ... calculate output tokens ...
    return (input_tokens * 0.000015) + (output_tokens * 0.000075)
```

---

## 3.3 Iteration Limits & Termination

### Why You Need Iteration Limits

Without limits, agents can loop forever:

```
[THINK] "I need more information"
[ACT] Call tool
[OBSERVE] Got same information as before
[THINK] "I need more information"  ← Repeats infinitely!
[ACT] Call tool (same call)
...
```

### Setting Effective Limits

```python
# Basic: Hard iteration limit
max_iterations = 10  # Stop after 10 loops, even if not done

# Better: Multiple safeguards
def should_continue(state, max_iterations=10, timeout_seconds=30):
    """Decide if agent should continue looping."""
    
    # Hard iteration limit
    if state["iteration"] >= max_iterations:
        return False, "Max iterations reached"
    
    # Timeout protection
    elapsed = (datetime.now() - state["started_at"]).total_seconds()
    if elapsed > timeout_seconds:
        return False, "Timeout reached"
    
    # Detect loops (calling same tool repeatedly)
    recent_calls = state["tool_calls"][-3:]
    if len(recent_calls) == 3:
        if all(c.tool_name == recent_calls[0].tool_name for c in recent_calls):
            # Called same tool 3 times in a row
            return False, "Likely infinite loop detected"
    
    return True, ""
```

### Smart Termination Signals

Instead of just a hard limit, look for signals that the agent is done:

```python
def detect_completion(response, state):
    """Has the agent actually finished?"""
    
    # Signal 1: Model says end_turn
    if response.stop_reason == "end_turn":
        return True, "Model ended naturally"
    
    # Signal 2: No tool calls (means answer is ready)
    tool_calls = [b for b in response.content if b.type == "tool_use"]
    if not tool_calls:
        return True, "No tools called (answer ready)"
    
    # Signal 3: Final answer marker
    text = "\n".join(
        b.text for b in response.content if hasattr(b, "text")
    )
    if "final answer:" in text.lower():
        return True, "Final answer marker detected"
    
    return False, ""
```

---

## 3.4 Error Recovery in Loops

### Common Errors and Recovery

```python
def robust_agent_loop(user_query: str):
    """Agent loop with comprehensive error handling."""
    messages = [{"role": "user", "content": user_query}]
    
    for iteration in range(10):
        try:
            # Call model with timeout
            response = client.messages.create(
                model="claude-opus-4-8",
                max_tokens=2048,
                messages=messages,
                timeout=10.0  # 10 second API timeout
            )
        except TimeoutError:
            # Recovery: Return best answer available
            return "I couldn't complete the full analysis due to timeout. " + \
                   "Here's what I found so far: [partial results]"
        except Exception as e:
            # Recovery: Log and retry once
            if iteration < 9:
                time.sleep(2)  # Back off before retry
                continue
            else:
                return f"Service error: {e}"
        
        # Process response normally
        tool_calls = [b for b in response.content if b.type == "tool_use"]
        
        if not tool_calls:
            for b in response.content:
                if hasattr(b, "text"):
                    return b.text
        
        messages.append({"role": "assistant", "content": response.content})
        
        # Execute tools with error handling
        tool_results = []
        for tool_call in tool_calls:
            try:
                result = execute_tool(tool_call.name, tool_call.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": result
                })
            except ValueError as e:
                # Input validation error — send back to model
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": f"Invalid input: {e}. Please try again with different parameters.",
                    "is_error": True
                })
            except Exception as e:
                # Unexpected error — inform model
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": f"Tool execution error: {e}",
                    "is_error": True
                })
        
        messages.append({"role": "user", "content": tool_results})
    
    return "Maximum iterations reached without final answer"
```

### When to Retry vs Give Up

```python
def decide_on_error(error_message: str, attempt: int, max_attempts: int = 3):
    """Should we retry or give up?"""
    
    # Always retry: transient network errors
    if "timeout" in error_message.lower():
        return True, "Transient network error"
    if "connection" in error_message.lower():
        return True, "Connection error"
    
    # Never retry: permanent errors
    if "invalid" in error_message.lower() or "not found" in error_message.lower():
        return False, "Permanent error"
    if "unauthorized" in error_message.lower() or "forbidden" in error_message.lower():
        return False, "Permission denied"
    
    # Retry once, then give up
    if attempt < max_attempts:
        return True, "Retrying"
    else:
        return False, "Max retries exceeded"
```

---

# Part 4: Implementation Patterns

## 4.1 Basic ReAct Agent

### The Simplest Version (Copy-Paste Ready)

```python
from anthropic import Anthropic
from datetime import datetime

client = Anthropic()

# Define your tools
TOOLS = [
    {
        "name": "search_web",
        "description": "Search the web for information",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "The search query"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "calculate",
        "description": "Perform mathematical calculations",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {"type": "string", "description": "Math expression (e.g., '2+2*3')"}
            },
            "required": ["expression"]
        }
    }
]

# System prompt with ReAct guidance
SYSTEM_PROMPT = """You are an intelligent assistant that reasons before acting.

When answering questions:
1. THINK about what information you need
2. ACT by calling tools if necessary
3. OBSERVE the results
4. REPEAT until you have enough information

Always show your thinking process. Format your response:

<thinking>
[Your reasoning about what to do next]
</thinking>

Then either call a tool or provide your final answer.

Available tools: search_web, calculate"""

def execute_tool(tool_name: str, tool_input: dict) -> str:
    """Execute a tool (mock implementation)."""
    if tool_name == "search_web":
        query = tool_input.get("query", "")
        return f"Search results for '{query}': [mock results about {query}]"
    elif tool_name == "calculate":
        expr = tool_input.get("expression", "")
        try:
            result = eval(expr)  # ⚠️ Only safe for math expressions!
            return str(result)
        except:
            return "Invalid expression"
    return "Unknown tool"

def basic_react_agent(user_query: str, verbose: bool = True) -> str:
    """Run a ReAct agent (basic version)."""
    
    messages = [{"role": "user", "content": user_query}]
    
    if verbose:
        print(f"\n{'='*60}")
        print(f"Query: {user_query}")
        print(f"{'='*60}\n")
    
    for iteration in range(10):
        if verbose:
            print(f"[Iteration {iteration + 1}]")
        
        # Call model
        response = client.messages.create(
            model="claude-opus-4-8",
            max_tokens=2048,
            system=SYSTEM_PROMPT,
            tools=TOOLS,
            messages=messages
        )
        
        # Extract and display thinking
        thinking_text = ""
        for block in response.content:
            if hasattr(block, "text"):
                thinking_text = block.text
                if verbose:
                    print(f"Thinking: {thinking_text[:200]}...")
        
        # Check if done
        if response.stop_reason == "end_turn":
            if verbose:
                print(f"\n✓ Agent finished.\n")
            return thinking_text
        
        # Extract tool calls
        tool_calls = [b for b in response.content if b.type == "tool_use"]
        
        if not tool_calls:
            if verbose:
                print(f"\n✓ No tools to call. Returning answer.\n")
            return thinking_text
        
        # Add assistant response to history
        messages.append({"role": "assistant", "content": response.content})
        
        # Execute tools
        tool_results = []
        for tool_call in tool_calls:
            if verbose:
                print(f"→ Calling {tool_call.name}({tool_call.input})")
            
            result = execute_tool(tool_call.name, tool_call.input)
            
            if verbose:
                print(f"← Result: {result[:100]}")
            
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tool_call.id,
                "content": result
            })
        
        # Add tool results to history
        messages.append({"role": "user", "content": tool_results})
    
    return "Max iterations reached"

# Test it
if __name__ == "__main__":
    answer = basic_react_agent(
        "What's 15% of 847? Show your reasoning."
    )
    print(f"\nFinal Answer: {answer}")
```

**Run this code and you'll see the full ReAct loop in action.**

---

## 4.2 Stateful Agent with Tracking

### Adding Observability

```python
from typing import TypedDict
from dataclasses import dataclass, field
import json

@dataclass
class ToolExecution:
    """Record of a single tool call."""
    iteration: int
    tool_name: str
    input: dict
    output: str
    success: bool
    latency_ms: float

class AgentStateDict(TypedDict):
    """Type hint for agent state."""
    iteration: int
    messages: list
    tool_executions: list[ToolExecution]
    reasoning_chain: str
    final_answer: str
    started_at: str

class ObservableReActAgent:
    """ReAct agent with detailed tracking."""
    
    def __init__(self, max_iterations: int = 10):
        self.max_iterations = max_iterations
        self.state: AgentStateDict = {
            "iteration": 0,
            "messages": [],
            "tool_executions": [],
            "reasoning_chain": "",
            "final_answer": "",
            "started_at": datetime.now().isoformat()
        }
    
    def run(self, user_query: str) -> AgentStateDict:
        """Execute agent and return complete state."""
        
        self.state["messages"] = [{"role": "user", "content": user_query}]
        
        for iteration in range(self.max_iterations):
            self.state["iteration"] = iteration
            
            # Call model
            response = client.messages.create(
                model="claude-opus-4-8",
                max_tokens=2048,
                system=SYSTEM_PROMPT,
                tools=TOOLS,
                messages=self.state["messages"]
            )
            
            # Capture reasoning
            for block in response.content:
                if hasattr(block, "text"):
                    self.state["reasoning_chain"] += f"\n[Iter {iteration}]\n{block.text}"
            
            # Check termination
            if response.stop_reason == "end_turn":
                for block in response.content:
                    if hasattr(block, "text"):
                        self.state["final_answer"] = block.text
                break
            
            # Extract tool calls
            tool_calls = [b for b in response.content if b.type == "tool_use"]
            if not tool_calls:
                for block in response.content:
                    if hasattr(block, "text"):
                        self.state["final_answer"] = block.text
                break
            
            # Add to messages
            self.state["messages"].append({"role": "assistant", "content": response.content})
            
            # Execute and track tools
            tool_results = []
            for tool_call in tool_calls:
                import time
                start = time.time()
                
                try:
                    result = execute_tool(tool_call.name, tool_call.input)
                    success = True
                except Exception as e:
                    result = str(e)
                    success = False
                
                latency = (time.time() - start) * 1000
                
                # Record execution
                execution = ToolExecution(
                    iteration=iteration,
                    tool_name=tool_call.name,
                    input=tool_call.input,
                    output=result,
                    success=success,
                    latency_ms=latency
                )
                self.state["tool_executions"].append(execution)
                
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_call.id,
                    "content": result
                })
            
            self.state["messages"].append({"role": "user", "content": tool_results})
        
        return self.state
    
    def get_metrics(self) -> dict:
        """Analyze agent performance."""
        executions = self.state["tool_executions"]
        return {
            "total_iterations": self.state["iteration"],
            "tools_called": len(executions),
            "successful_calls": sum(1 for e in executions if e.success),
            "failed_calls": sum(1 for e in executions if not e.success),
            "total_latency_ms": sum(e.latency_ms for e in executions),
            "avg_tool_latency_ms": sum(e.latency_ms for e in executions) / len(executions) if executions else 0
        }
    
    def get_reasoning_trace(self) -> str:
        """Full THINK-ACT-OBSERVE narrative."""
        return self.state["reasoning_chain"]
    
    def get_tool_execution_log(self) -> list[dict]:
        """Detailed log of each tool call."""
        return [
            {
                "iteration": e.iteration,
                "tool": e.tool_name,
                "input": e.input,
                "output": e.output[:200],  # Truncate for readability
                "success": e.success,
                "latency_ms": f"{e.latency_ms:.1f}"
            }
            for e in self.state["tool_executions"]
        ]

# Usage
agent = ObservableReActAgent()
final_state = agent.run("Calculate: (100 + 200) / 2")

print("=== METRICS ===")
for key, value in agent.get_metrics().items():
    print(f"{key}: {value}")

print("\n=== TOOL EXECUTION LOG ===")
for entry in agent.get_tool_execution_log():
    print(json.dumps(entry, indent=2))

print("\n=== REASONING TRACE ===")
print(agent.get_reasoning_trace())
```

---

## 4.3 Advanced Patterns

### Pattern 1: Branching (Multiple Parallel Paths)

```python
def branching_agent(user_query: str):
    """Agent that explores multiple paths and picks the best."""
    
    # Step 1: Initial analysis
    messages = [{"role": "user", "content": user_query}]
    response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=messages,
        system="Identify 3 different approaches to solving this problem. Just list them, no tools."
    )
    
    approaches = response.content[0].text  # Gets 3 approaches
    
    # Step 2: Execute each approach in parallel
    results = []
    for approach_num in range(3):
        approach_messages = [
            {"role": "user", "content": user_query},
            {"role": "assistant", "content": f"I'll use approach {approach_num + 1}"},
        ]
        
        # Run agent with this approach
        result = basic_react_agent(user_query, verbose=False)  # Simplified
        results.append({
            "approach": approach_num + 1,
            "result": result
        })
    
    # Step 3: Compare and pick best
    comparison = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"Compare these solutions and pick the best:\n{json.dumps(results)}"
        }]
    )
    
    return comparison.content[0].text
```

### Pattern 2: Hierarchical Planning

```python
def hierarchical_agent(user_query: str):
    """Agent that decomposes into subtasks."""
    
    # Step 1: Decompose
    messages = [{"role": "user", "content": user_query}]
    decompose_response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=messages,
        system="Break this task into 3-5 concrete subtasks. Be specific."
    )
    
    subtasks_text = decompose_response.content[0].text
    # Parse subtasks (in real code, use structured output)
    subtasks = subtasks_text.split("\n")[:5]
    
    # Step 2: Execute subtasks
    subtask_results = []
    for subtask in subtasks:
        if not subtask.strip():
            continue
        result = basic_react_agent(subtask, verbose=False)
        subtask_results.append({"task": subtask, "result": result})
    
    # Step 3: Integrate
    integration_response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"Integrate these subtask results into a final answer:\n{json.dumps(subtask_results)}"
        }]
    )
    
    return integration_response.content[0].text
```

---

## 4.4 Common Implementation Mistakes

### Mistake 1: Not Feeding Tool Results Back

```python
# ❌ WRONG
messages.append({"role": "assistant", "content": response.content})
# Forgot to add tool results!
# Model doesn't see what the tools returned

# ✅ CORRECT
messages.append({"role": "assistant", "content": response.content})
messages.append({"role": "user", "content": tool_results})  # Add this!
```

**Why it matters:** The model needs to see tool results to adapt its behavior.

### Mistake 2: Not Checking stop_reason

```python
# ❌ WRONG
response = client.messages.create(...)
# Assumes there are always tool calls
tool_calls = [b for b in response.content if b.type == "tool_use"]
# Crashes if no tool calls exist

# ✅ CORRECT
if response.stop_reason == "end_turn":
    return extract_answer(response)  # Model is done
tool_calls = [b for b in response.content if b.type == "tool_use"]
if not tool_calls:
    return extract_answer(response)  # No tools needed
```

**Why it matters:** stop_reason tells you why the response ended. Always check it.

### Mistake 3: Infinite Loops

```python
# ❌ WRONG
while True:  # No exit condition!
    response = client.messages.create(...)
    messages.append(...)
    # If model keeps calling tools, this loops forever

# ✅ CORRECT
for iteration in range(max_iterations):  # Hard limit
    response = client.messages.create(...)
    if response.stop_reason == "end_turn":
        break
    if iteration >= max_iterations - 1:
        break  # Safety exit
    messages.append(...)
```

**Why it matters:** Protects against runaway costs and bad user experience.

### Mistake 4: Not Handling Tool Errors

```python
# ❌ WRONG
result = execute_tool(tool_name, tool_input)
# If execute_tool throws, whole agent crashes

# ✅ CORRECT
try:
    result = execute_tool(tool_name, tool_input)
except Exception as e:
    result = f"Tool error: {e}"
    # Tool results with errors are still sent back to model
    # Model can see the error and adapt
```

**Why it matters:** Graceful error handling lets the model recover intelligently.

### Mistake 5: Tool Results Not Visible to Model

```python
# ❌ WRONG
tool_result = execute_tool(...)
# Forgot to format as tool_result message
messages.append({"role": "assistant", "content": tool_result})  # Wrong!

# ✅ CORRECT
tool_results = [{
    "type": "tool_result",
    "tool_use_id": tool_call.id,  # Must match the tool_use block id
    "content": tool_result
}]
messages.append({"role": "user", "content": tool_results})
```

**Why it matters:** Tool results must be properly formatted for the model to parse them.

---

# Part 5: Production Considerations

## 5.1 Debugging Agents

### The Three-Level Debug Approach

#### Level 1: Check the Messages

```python
def debug_messages(messages: list):
    """Inspect what the model sees."""
    for i, msg in enumerate(messages):
        print(f"\n--- Message {i} ---")
        print(f"Role: {msg['role']}")
        if isinstance(msg['content'], str):
            print(f"Content: {msg['content'][:200]}")
        else:
            for block in msg['content']:
                if hasattr(block, 'text'):
                    print(f"Text: {block.text[:200]}")
                if hasattr(block, 'type'):
                    if block.type == 'tool_use':
                        print(f"Tool: {block.name}({block.input})")

# Use it
debug_messages(state["messages"])
# Output shows exactly what the model is seeing
```

#### Level 2: Check the Reasoning Chain

```python
def analyze_reasoning(reasoning_chain: str):
    """Look for red flags in the model's thinking."""
    
    red_flags = [
        ("Uncertain", "model is unsure"),
        ("I don't know", "model admits lack of knowledge"),
        ("But I can't", "model sees limitations"),
        ("Let me try", "model is recovering from error"),
    ]
    
    for flag, meaning in red_flags:
        if flag.lower() in reasoning_chain.lower():
            print(f"⚠️  Found: {flag} ({meaning})")
    
    # Count iteration patterns
    iterations = reasoning_chain.split("[Iter")
    print(f"Total iterations: {len(iterations)}")
```

#### Level 3: Check Tool Execution

```python
def debug_tool_calls(tool_executions: list[ToolExecution]):
    """Inspect what tools were called and what happened."""
    
    for exe in tool_executions:
        status = "✓" if exe.success else "✗"
        print(f"{status} [{exe.iteration}] {exe.tool_name}")
        print(f"   Input: {exe.input}")
        print(f"   Output: {exe.output[:100]}")
        print(f"   Latency: {exe.latency_ms:.0f}ms")
        print()
    
    # Detect patterns
    tool_names = [e.tool_name for e in tool_executions]
    from collections import Counter
    print(f"Tool frequency: {dict(Counter(tool_names))}")
    
    if len(tool_names) != len(set(tool_names)):
        print("⚠️  Duplicate tool calls detected")
```

### Debugging Workflow

1. **Reproduce the problem** — Get a specific failing query
2. **Enable verbose output** — Print messages and reasoning
3. **Check all three levels** — Messages → Reasoning → Tools
4. **Identify the bottleneck** — Is it the prompt, the tools, or the loop logic?
5. **Fix incrementally** — Change one thing, test again

---

## 5.2 Monitoring & Observability

### Key Metrics to Track

```python
class AgentMetrics:
    """Collect and analyze agent performance."""
    
    def __init__(self):
        self.runs = []  # List of agent runs
    
    def record_run(self, state: dict, success: bool, latency_sec: float):
        """Record metrics from a single agent run."""
        self.runs.append({
            "timestamp": datetime.now(),
            "success": success,
            "latency_sec": latency_sec,
            "iterations": state["iteration"],
            "tools_called": len(state["tool_executions"]),
            "final_answer_length": len(state["final_answer"]),
            "reasoning_length": len(state["reasoning_chain"])
        })
    
    def get_summary(self) -> dict:
        """Aggregate metrics."""
        if not self.runs:
            return {}
        
        latencies = [r["latency_sec"] for r in self.runs]
        successes = [r["success"] for r in self.runs]
        iterations = [r["iterations"] for r in self.runs]
        
        return {
            "total_runs": len(self.runs),
            "success_rate": sum(successes) / len(successes),
            "avg_latency_sec": sum(latencies) / len(latencies),
            "p95_latency_sec": sorted(latencies)[int(len(latencies) * 0.95)],
            "avg_iterations": sum(iterations) / len(iterations),
            "max_iterations": max(iterations)
        }

# Usage
metrics = AgentMetrics()
for i in range(100):  # Run agent 100 times
    start = time.time()
    state = agent.run(user_query)
    latency = time.time() - start
    metrics.record_run(state, success=True, latency_sec=latency)

summary = metrics.get_summary()
print(f"Success rate: {summary['success_rate']:.1%}")
print(f"P95 latency: {summary['p95_latency_sec']:.2f}s")
```

### Alerting

```python
def check_agent_health(metrics_summary: dict):
    """Alert if agent is degrading."""
    
    alerts = []
    
    if metrics_summary.get("success_rate", 1) < 0.9:
        alerts.append("⚠️  Success rate below 90%")
    
    if metrics_summary.get("p95_latency_sec", 0) > 30:
        alerts.append("⚠️  P95 latency above 30s")
    
    if metrics_summary.get("max_iterations", 0) >= 10:
        alerts.append("⚠️  Agent hitting iteration limits")
    
    if alerts:
        send_alert_to_team(alerts)
    
    return alerts
```

---

## 5.3 Cost Management

### Tracking Costs

```python
class CostTracker:
    """Monitor spending on LLM API calls."""
    
    # Pricing as of 2025
    PRICING = {
        "claude-opus-4-8": {"input": 15e-6, "output": 75e-6},
        "claude-sonnet-4-6": {"input": 3e-6, "output": 15e-6},
        "claude-haiku-4-5": {"input": 0.8e-6, "output": 4e-6},
    }
    
    def __init__(self):
        self.calls = []
    
    def record_call(self, model: str, input_tokens: int, output_tokens: int):
        """Record an API call."""
        pricing = self.PRICING.get(model, {"input": 0, "output": 0})
        cost = (input_tokens * pricing["input"]) + (output_tokens * pricing["output"])
        
        self.calls.append({
            "model": model,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "cost": cost,
            "timestamp": datetime.now()
        })
    
    def get_total_cost(self, model: str = None) -> float:
        """Total cost of all calls (or specific model)."""
        calls = self.calls
        if model:
            calls = [c for c in calls if c["model"] == model]
        return sum(c["cost"] for c in calls)
    
    def estimate_monthly_cost(self, daily_queries: int, cost_per_query: float) -> float:
        """Project monthly cost."""
        return daily_queries * 30 * cost_per_query

# Usage
tracker = CostTracker()
for _ in range(100):
    # Simulate agent runs
    input_tokens = 500
    output_tokens = 250
    tracker.record_call("claude-opus-4-8", input_tokens, output_tokens)

total_cost = tracker.get_total_cost()
print(f"Total cost: ${total_cost:.2f}")

projected = tracker.estimate_monthly_cost(daily_queries=1000, cost_per_query=total_cost/100)
print(f"Projected monthly: ${projected:.0f}")
```

### Cost Optimization

```python
def optimize_for_cost(agent_state: dict) -> str:
    """Decide which model to use based on task complexity."""
    
    iterations = agent_state["iteration"]
    reasoning_complexity = len(agent_state["reasoning_chain"])
    tools_called = len(agent_state["tool_executions"])
    
    # Simple heuristic
    if iterations <= 2 and tools_called <= 1:
        return "claude-haiku-4-5"  # Cheapest
    elif iterations <= 5:
        return "claude-sonnet-4-6"  # Middle
    else:
        return "claude-opus-4-8"  # Most capable
```

---

## 5.4 Reliability & Resilience

### Retry Logic

```python
import time
from functools import wraps

def with_retry(max_attempts: int = 3, backoff_seconds: float = 1.0):
    """Decorator to retry agent runs."""
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt < max_attempts - 1:
                        wait_time = backoff_seconds * (2 ** attempt)
                        print(f"Attempt {attempt + 1} failed, retrying in {wait_time}s")
                        time.sleep(wait_time)
                    else:
                        raise
            return None
        return wrapper
    return decorator

@with_retry(max_attempts=3, backoff_seconds=1.0)
def resilient_agent_run(user_query: str) -> str:
    agent = ObservableReActAgent()
    state = agent.run(user_query)
    return state["final_answer"]
```

### Circuit Breaker Pattern

```python
class CircuitBreaker:
    """Prevent cascading failures."""
    
    def __init__(self, failure_threshold: int = 5, recovery_timeout_sec: int = 60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout_sec = recovery_timeout_sec
        self.last_failure_time = None
        self.is_open = False
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        if self.failure_count >= self.failure_threshold:
            self.is_open = True
            print(f"🔴 Circuit breaker OPEN (failures: {self.failure_count})")
    
    def record_success(self):
        self.failure_count = 0
        self.is_open = False
        print(f"🟢 Circuit breaker CLOSED")
    
    def allow_request(self) -> bool:
        if not self.is_open:
            return True
        
        # Check if recovery period has elapsed
        if self.last_failure_time:
            elapsed = (datetime.now() - self.last_failure_time).total_seconds()
            if elapsed > self.recovery_timeout_sec:
                print(f"🟡 Circuit breaker attempting recovery...")
                self.is_open = False
                self.failure_count = 0
                return True
        
        return False

# Usage
circuit = CircuitBreaker(failure_threshold=3)

for query in user_queries:
    if not circuit.allow_request():
        return "Service temporarily unavailable. Please try again later."
    
    try:
        result = agent.run(query)
        circuit.record_success()
    except Exception as e:
        circuit.record_failure()
        if circuit.is_open:
            return "Service unavailable"
```

---

# Part 6: Interview Q&A

## 6.1 Core Concept Questions

### Q1: Explain ReAct in One Sentence

**Strong Answer:**
> ReAct is a pattern where the model interleaves explicit reasoning steps with tool calls, letting it observe results and adapt rather than planning everything upfront. This makes reasoning auditable and enables error recovery.

**Why this works:** Concise, explains the mechanism, and highlights the benefit.

---

### Q2: Why Is ReAct Better Than Just Calling Tools?

**Strong Answer:**
> With pure tool calling, the model decides what tools to call and executes them, but you don't see the reasoning — if it makes a mistake, you can't tell why. With ReAct, the model's thinking is explicit (THINK step), so you can audit it. Also, when tools return unexpected results, the model can see them (OBSERVE) and adapt (next THINK) instead of getting stuck. ReAct adds auditability and adaptability.

**Why this works:** Compares directly, explains both benefits (auditability and adaptability), and gives concrete examples.

---

### Q3: When Would You Use ReAct vs Chain-of-Thought?

**Strong Answer:**
> Chain-of-Thought (CoT) is for tasks where the model needs to *think through* the problem internally — math, logic puzzles, reasoning about facts it already knows. You don't need to call external tools.
>
> ReAct is for tasks where the model needs *information from the external world* that it doesn't have access to — searching the web, querying databases, calling APIs. ReAct combines thinking with tool calls.
>
> Simple rule: Use CoT for reasoning. Use ReAct for reasoning + information retrieval.

**Why this works:** Clear distinction with memorable rule, plus examples.

---

### Q4: What Happens If a Tool Call Fails?

**Strong Answer:**
> The tool returns an error message (e.g., "Database connection timeout"). This error is sent back to the model as a tool_result, which the model sees in its next THINK step. The model can then decide: retry the tool with different parameters, try a different tool, or use cached/fallback data. This is why ReAct is resilient — the model has a chance to recover from tool failures rather than the whole agent crashing.

**Why this works:** Explains the mechanism, shows how it enables recovery, and highlights the advantage.

---

## 6.2 Implementation Questions

### Q5: Walk Me Through Your Agent Loop Code

**Strong Answer:**
> My agent loop has three main components:
>
> 1. **Initialize:** Start with user query in messages list
> 2. **Loop (up to max_iterations):**
>    - Call model with messages + tools
>    - Check stop_reason (if "end_turn", return answer)
>    - Extract any tool calls from response
>    - If no tool calls, return the text as final answer
>    - Otherwise, execute each tool and collect results
>    - Add assistant response + tool results to messages (important: this becomes context for next call)
> 3. **Return:** Either the final answer or timeout message
>
> Key safeguards:
> - Hard iteration limit (protects from infinite loops)
> - Check stop_reason (respects when model is done)
> - Add results back to messages (enables model to learn)
> - Error handling in tool execution (graceful failures)

**Why this works:** Shows you've built this before, explains the flow, and highlights critical details.

---

### Q6: How Do You Handle Tool Errors?

**Strong Answer:**
> I wrap tool execution in try-catch. If a tool fails:
> - I capture the error message
> - Send it back to the model as a tool_result with is_error=true
> - The model sees the error in its next THINK step
> - The model can retry with different parameters or try a different tool
>
> I distinguish between:
> - Transient errors (timeout, connection) → retry automatically
> - Permanent errors (invalid input) → send to model to decide
>
> This way, the model can recover intelligently rather than the agent crashing.

**Why this works:** Shows nuance (different types of errors get different handling), and explains the philosophy (model decides, not hardcoded logic).

---

### Q7: What's the Max Iterations Parameter? How Do You Set It?

**Strong Answer:**
> Max iterations is a safety limit — if the agent hasn't finished after N loops, we stop and return the best answer we have. It prevents:
> - Infinite loops (agent calling same tool repeatedly)
> - Runaway costs (each iteration costs tokens)
> - Poor user experience (waiting forever)
>
> I set it based on task complexity:
> - Simple lookup: 3-5 iterations (should find answer quickly)
> - Complex research: 10-15 iterations (may need multiple searches)
> - Open-ended tasks: 20 iterations (lots of reasoning)
>
> In production, I also add timeout checks (absolute time limit) and loop detection (same tool called 3x in a row → stop).

**Why this works:** Explains the purpose, shows it's not arbitrary, and adds production-level sophistication.

---

## 6.3 Production Questions

### Q8: How Would You Debug an Agent That's Hallucinating?

**Strong Answer:**
> Hallucination typically means the model is making up information instead of using tool results. Debug steps:
>
> 1. **Check messages:** Is the tool result actually being sent back to the model?
>    ```python
>    debug_messages(state["messages"])  # Shows what the model sees
>    ```
>
> 2. **Check reasoning:** Does the model's THINK step reference the tool result, or is it ignoring it?
>    ```python
>    # If model says "The population is X" but tool returned Y, 
>    # model is hallucinating
>    ```
>
> 3. **Check tool output:** Is the tool returning data in a format the model understands? Maybe the JSON is malformed.
>
> 4. **Check prompt:** Maybe the system prompt is encouraging hallucination. Change it to: "Use tool results. Don't make up information."
>
> Root causes:
> - Tool results not being sent back (implementation bug)
> - Tool returning confusing format (tool interface issue)
> - Model not trusting tool results (prompt issue)

**Why this works:** Structured debugging approach, shows you've seen this problem, and gives concrete fixes.

---

### Q9: Estimate the Cost of Running 1000 Agent Queries Using Claude Opus

**Strong Answer:**
> Let me estimate:
>
> **Per-query costs:**
> - Input: ~500 tokens (user query + system prompt)
> - Output: ~250 tokens (final answer)
> - Tool calls: ~3 tools × 2 calls each = 6 calls
> - Each tool call adds: ~100 tokens (input) + 100 tokens (output) = 200 tokens
> - Total: ~500 + 250 + (6 × 200) = 1,950 tokens input + 1,300 tokens output
>
> **Claude Opus pricing:**
> - Input: $0.000015 per token
> - Output: $0.000075 per token
>
> **Per query:**
> - Input cost: 1,950 × $0.000015 = $0.029
> - Output cost: 1,300 × $0.000075 = $0.098
> - Total: ~$0.13 per query
>
> **For 1000 queries:**
> - Total: 1,000 × $0.13 = $130
>
> **Optimizations to reduce:**
> - Use Sonnet for 30% of queries (cheaper) → saves $30
> - Prompt caching for system prompt → saves 50% on first 100 tokens → saves $10
> - Reduce average tool calls (smarter prompts) → could save $20+
>
> **Realistic estimate: $70-100 for 1000 queries**

**Why this works:** Shows calculation skills, realistic estimates, and optimization awareness.

---

### Q10: An Agent Is Hitting the Max Iterations Limit. How Do You Diagnose?

**Strong Answer:**
> If the agent consistently hits max iterations, it means it's not finding a complete answer. Diagnose with:
>
> 1. **Check the reasoning chain:**
>    ```python
>    print(agent_state["reasoning_chain"])
>    ```
>    Is it repeating the same pattern? Or exploring different approaches?
>
> 2. **Check tool execution:**
>    - Is the agent calling the same tool over and over?
>    - Are tools returning useful data or empty results?
>    - Is latency high (slow tools = fewer iterations)?
>
> 3. **Check the tools themselves:**
>    - Does the agent have the right tools for the task?
>    - Are tool descriptions clear?
>    - Are tools returning data in the expected format?
>
> **Fixes:**
> - **Better prompt:** Explicitly guide the agent (e.g., "Try 2 different search strategies, not just retrying the same search")
> - **Better tools:** Add tools that directly answer the question (don't make it multi-step)
> - **Increase iterations:** If the agent is making progress but slowly, just increase the limit
> - **Hybrid approach:** Use search + LLM to synthesize, instead of making agent loop endlessly

**Why this works:** Shows you understand the tool-agent relationship and have solutions ready.

---

## Summary: Key Takeaways

### ReAct Core
- **Pattern:** THINK → ACT → OBSERVE → repeat
- **Benefit:** Explicit reasoning + error recovery
- **Use case:** Information-seeking tasks with tool calls

### Agentic Loops
- **Structure:** Message loop with state
- **Safety:** Iteration limits + timeout + error handling
- **Tracking:** Capture reasoning and tool calls for debugging

### Implementation
- **Basic:** Simple loop calling model → extracting tools → executing → feeding back
- **Advanced:** Branching, hierarchical planning, stateful tracking
- **Critical:** Always add tool results back to messages

### Production
- **Debug:** Check messages → reasoning → tools
- **Monitor:** Track success rate, latency, iterations
- **Optimize:** Cost tracking, circuit breakers, caching

---

*End of Complete ReAct + Agentic Loops Master Guide*

**This guide covers everything from mental models through production deployment.**  
**Use it as a reference when building, interviewing, or teaching agents.**
