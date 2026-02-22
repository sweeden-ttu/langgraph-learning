# LangGraph — Example 1: A Gentle Introduction

**A Step-by-Step Tutorial**  
*Instructor: A. Namin*

---

## Table of Contents

1. [Overview](#1-overview)
2. [What Is LangGraph?](#2-what-is-langgraph)
3. [LangGraph vs LangChain](#3-langgraph-vs-langchain)
4. [Core Concepts](#4-core-concepts-in-langgraph)
5. [Best Practice: Bottom-Up Development](#5-best-practice-bottom-up-development)
6. [Tutorial Example: Customer Feedback Processor](#6-tutorial-example-customer-feedback-processor)
7. [Step-by-Step Implementation](#7-step-by-step-implementation)
8. [Executing and Monitoring the Graph](#8-executing-and-monitoring-the-graph)
9. [Advanced: Appending vs Overwriting State](#9-advanced-appending-vs-overwriting-state)
10. [Custom Operators for Complex Types](#10-custom-operators-for-complex-types)
11. [Parallel Node Execution](#11-parallel-node-execution)
12. [Advanced LLM Integration](#12-advanced-llm-integration)
13. [Key Takeaways](#13-key-takeaways)
14. [Next Steps and Resources](#14-next-steps-and-resources)
15. [Hands-On Exercises](#15-hands-on-exercises)
16. [Debugging Tips](#16-debugging-tips)
17. [Complete Code Reference](#17-complete-code-reference)
18. [References](#18-references)

---

## 1. Overview

This tutorial provides a gentle introduction to **LangGraph**: an orchestration framework for building complex agent systems with predictable, graph-based workflows. You will learn core concepts, build a Customer Feedback Processor example, and explore state management, conditional routing, and parallel execution.

---

## 2. What Is LangGraph?

### Definition

LangGraph is an **orchestration framework** for building complex **agent** systems.

### Key Characteristics

- **Low-level control:** More granular than LangChain agents.
- **Deterministic flows:** Pre-defined execution paths guaranteed every time.
- **Graph-based:** Workflows represented as directed graphs with nodes and edges.
- **State management:** Structured state shared across all nodes.

---

## 3. LangGraph vs LangChain

### LangChain Agents

- Dynamic reasoning at runtime.
- Plans change based on observations.
- Less control over execution flow.
- Good for exploratory tasks.

### LangGraph

- Pre-defined workflow structure.
- Guaranteed execution path.
- Full control over flow; production-ready determinism.

### Key Insight

Use LangGraph when you need **predictable, controlled workflows** that must follow specific business logic every time.

---

## 4. Core Concepts in LangGraph

| # | Concept | Description |
| -- | ------- | ----------- |
| 1 | **State** | A shared data structure (TypedDict) that flows through the graph. |
| 2 | **Nodes** | Functions that process and update the state. |
| 3 | **Edges** | Connections defining flow between nodes. |
| 4 | **Conditional Edges** | Decision points that route to different nodes based on conditions. |
| 5 | **Graph Builder** | Tool to construct and compile the workflow. |

---

## 5. Best Practice: Bottom-Up Development

### Step-by-Step Workflow

1. **Draw the graph on paper first.**
2. Define nodes using `add_node()`.
3. Create edges with `add_edge()`.
4. Add conditional edges if needed.
5. Define the State class.
6. Implement node functions.
7. Compile and visualize.
8. Test and iterate.

### Pro Tip

Always start with a visual diagram. It helps you think through the logic before writing code.

---

## 6. Tutorial Example: Customer Feedback Processor

### Business Problem

Process social media comments to identify **questions** vs **compliments** and route them appropriately.

### Workflow Overview

```text
START
  ↓
Extract Content
  ↓
Route (Question / Compliment)
  ↓  Question   → Run Question Code
  ↓  Compliment → Run Compliment Code
  ↓
Beautify
  ↓
END
```

### Input

API payload with customer remarks, timestamps, social media channel, etc.

---

## 7. Step-by-Step Implementation

### 7.1 Creating Nodes

```python
from langgraph.graph import StateGraph

graph_builder = StateGraph(State)

# Add all nodes
graph_builder.add_node("extract_content", extract_content)
graph_builder.add_node("run_question_code", run_question_code)
graph_builder.add_node("run_compliment_code", run_compliment_code)
graph_builder.add_node("beautify", beautify)
```

**Notes:**

- `START` and `END` nodes are built-in (no need to create them).
- First argument = node name (string); second argument = Python function.
- Names can differ, but keeping them identical improves readability.

---

### 7.2 Creating Standard Edges

```python
from langgraph.graph import END, START

# Define edges (connections between nodes)
graph_builder.add_edge(START, "extract_content")
graph_builder.add_edge("run_question_code", "beautify")
graph_builder.add_edge("run_compliment_code", "beautify")
graph_builder.add_edge("beautify", END)
```

**What are edges?**  
Edges define the flow of execution from one node to another. They create the graph structure.

**Remember:** Use **node names** (strings), not function names, when adding edges.

---

### 7.3 Creating Conditional Edges

```python
graph_builder.add_conditional_edges(
  "extract_content",           # Source node
  route_question_or_compliment, # Routing function
  {
    "compliment": "run_compliment_code",
    "question": "run_question_code",
  },
)
```

**Three required arguments:**

1. **Source node:** Where the conditional check happens.
2. **Routing function:** Contains conditional logic; returns a string.
3. **Route mapping:** Dictionary mapping return values to target nodes.

**Key point:** The routing function must return one of the keys in the mapping dictionary.

---

### 7.4 Defining the State Class

```python
from typing_extensions import TypedDict

class State(TypedDict):
    text: str                    # Extracted content
    answer: str                  # Final response
    payload: dict[str, list]      # Input data
```

**State purpose:**

- **Centralized data storage:** All variables accessible across nodes.
- **Type safety:** TypedDict provides type hints.
- **Data flow:** Information passes between nodes via state.
- **Immutability:** Each node returns updated values; it doesn’t mutate state directly.

**Best practice:** Define all variables you’ll need upfront in the State class.

---

### 7.5 Implementing Node Functions (Part 1)

**Extract Content node:**

```python
def extract_content(state: State):
    # Access payload from state; extract customer_remark field
    return {"text": state["payload"][0]["customer_remark"]}
```

**Routing function:**

```python
def route_question_or_compliment(state: State):
    if "?" in state["text"]:
        return "question"
    else:
        return "compliment"
```

**Pattern:** Every node function:

- Takes `state: State` as parameter.
- Returns a dictionary with updated state variables.
- Accesses existing state via `state["variable_name"]`.

---

### 7.6 Implementing Node Functions (Part 2)

**Action nodes:**

```python
def run_compliment_code(state: State):
    return {"answer": "Thanks for the compliment."}

def run_question_code(state: State):
    return {"answer": "Wow nice question."}
```

**Beautify node:**

```python
def beautify(state: State):
    # Access current answer and modify it
    return {"answer": state["answer"] + " beautified"}
```

**Note:** By default, returning a dictionary with an existing key **overwrites** that variable in the state. Appending is covered in [§9](#9-advanced-appending-vs-overwriting-state).

---

### 7.7 Compiling and Visualizing

```python
# Compile the graph
graph = graph_builder.compile()

# Visualize the graph
from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))
```

**Visual indicators:**

- **Solid edges:** Always executed in the workflow.
- **Dotted edges:** Conditional — only one branch executes.

**Pro tip:** Always visualize your graph before running to verify the structure matches your design.

---

## 8. Executing and Monitoring the Graph

### 8.1 Using `invoke()`

```python
result = graph.invoke({
    "payload": [{
        "time_of_comment": "20-01-2025",
        "customer_remark": "I hate this.",
        "social_media_channel": "facebook",
        "number_of_likes": 100,
    }]
})

print(result)
```

**Example output:**

```python
{
    'text': 'I hate this.',
    'answer': 'Thanks for the compliment. beautified',
    'payload': [...]
}
```

`invoke()` returns the **complete final state** after graph execution.

---

### 8.2 Monitoring Execution with `stream()`

```python
for step in graph.stream({
    "payload": [{
        "customer_remark": "I hate this.",
        ...
    }]
}):
    print(step)
```

**Example output (step-by-step):**

```python
{'extract_content': {'text': 'I hate this.'}}
{'run_compliment_code': {'answer': 'Thanks for the compliment.'}}
{'beautify': {'answer': 'Thanks for the compliment. beautified'}}
```

**Use cases for `stream()`:**

- Showing progress bars in UI.
- Debugging node-by-node execution.
- Real-time updates to users.

---

## 9. Advanced: Appending vs Overwriting State

### The Problem

By default, updating a state variable **overwrites** its value. Sometimes you want to **append** instead.

### Solution: Annotated Types

```python
from typing import Annotated
import operator

class State(TypedDict):
    text: str
    # Changed from str to list with operator.add
    answer: Annotated[list, operator.add]
    payload: dict[str, list]
```

**Annotated format:** `Annotated[data_type, operator]`

- `operator.add` works with lists, strings, and numbers.
- Each return value gets **appended** instead of replacing.

---

### Updating Functions for Appending

**Modified node functions:**

```python
def run_compliment_code(state: State):
    return {"answer": ["Thanks for the compliment."]}

def run_question_code(state: State):
    return {"answer": ["Wow nice question."]}
```

**Modified beautify function:**

```python
def beautify(state: State):
    last_answer = state["answer"][-1]
    return {"answer": [last_answer + " beautified"]}
```

**Result:** `answer` now contains both intermediate and final values:

```python
['Thanks for the compliment.', 'Thanks for the compliment. beautified']
```

---

## 10. Custom Operators for Complex Types

### Why operator.add Isn't Enough

`operator.add` doesn’t work with dictionaries. For complex data structures, you need custom merge logic.

### Creating a Custom Operator

```python
def merge_dicts(dict1, dict2):
    return {**dict1, **dict2}

class State(TypedDict):
    text: str
    answer: Annotated[dict, merge_dicts]  # Custom operator
    payload: dict[str, list]
```

**Power tip:** You can create custom operators for any complex merge logic — nested dictionaries, custom objects, etc.

---

### Using Custom Operators: Example

**Updated node functions:**

```python
def run_compliment_code(state: State):
    return {"answer": {"temp_answer": "Thanks for the compliment."}}

def beautify(state: State):
    return {
        "answer": {
            "final_beautified_answer":
                state["answer"]["temp_answer"] + " beautified"
        }
    }
```

**Example final output:**

```python
{
    'answer': {
        'temp_answer': 'Thanks for the compliment.',
        'final_beautified_answer': 'Thanks for the compliment. beautified'
    }
}
```

---

## 11. Parallel Node Execution

### Use Case

New requirement: tag the type of customer remark (packaging, sustainability, medical) while determining if it’s a question or compliment.

### Enhanced Workflow

```text
START
  ↓
Extract Content
  ↓ (parallel execution)
  ├─→ Tag Query  ─┐
  └─→ Route      ─┼→ Beautify → END
      ↓ Question   → Run Question Code   ↑
      ↓ Compliment → Run Compliment Code ↑
```

**Benefit:** Nodes execute concurrently in the same superstep for efficiency.

---

### Implementing Parallel Execution

**Add new node and edge:**

```python
graph_builder.add_node("tag_query", tag_query)
graph_builder.add_edge("tag_query", "beautify")
```

**Tag Query function:**

```python
def tag_query(state: State):
    if "package" in state["text"]:
        return {"tag": "Packaging"}
    elif "price" in state["text"]:
        return {"tag": "Pricing"}
    else:
        return {"tag": "General"}
```

**Update State:**

```python
class State(TypedDict):
    text: str
    tag: str  # New variable
    answer: Annotated[dict, merge_dicts]
    payload: dict[str, list]
```

---

### Using Parallel Results

**Updated Beautify function:**

```python
def beautify(state: State):
    return {
        "answer": {
            "final_beautified_answer":
                state["answer"]["temp_answer"] +
                f' I will pass it to the {state["tag"]} Department'
        }
    }
```

**Example final output:**

```python
{
    'tag': 'General',
    'answer': {
        'final_beautified_answer':
            'Thanks for the compliment. I will pass it to the General Department'
    }
}
```

---

## 12. Advanced LLM Integration

### Enhanced Routing with LLM

```python
from langchain_openai import AzureChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = AzureChatOpenAI(
    deployment_name="gpt-4o",
    model_name="gpt-4o",
    temperature=0.1,
)

template = """
I have a piece of text: {text}.
Tell me whether it is a 'compliment' or a 'question'.
"""

prompt = ChatPromptTemplate([("user", template)])
chain = prompt | llm | StrOutputParser()

def route_question_or_compliment(state: State):
    response = chain.invoke({"text": state["text"]})
    return response
```

---

## 13. Key Takeaways

### Core Principles

1. **Start with a diagram** — visualize before coding.
2. **State is central** — all data flows through it.
3. **Nodes are functions** — they transform state.
4. **Edges define flow** — control execution order.
5. **Conditional edges** — enable decision making.
6. **Operators control merging** — append vs overwrite.

### When to Use LangGraph

- Production workflows requiring deterministic execution.
- Complex multi-step AI agent workflows.
- When you need full control over execution flow.
- Building reliable, reproducible AI systems.

---

## 14. Next Steps and Resources

### What We Covered

- Basic graph construction.
- State management and conditional routing.
- Custom operators.
- Parallel execution.
- LLM integration.

### Next Steps

- **Part 2:** RAG-based workflows with LangGraph.
- Using the Send API for advanced map-reduce patterns.
- Building production-ready AI agents.
- Implementing human-in-the-loop workflows.

### Resources

- **GitHub:** V-Sher/LangGraphTutorial  
- **Documentation:** [docs.langchain.com](https://docs.langchain.com)  
- **Original article:** [levelup.gitconnected.com — Gentle Introduction to LangGraph](https://levelup.gitconnected.com/gentle-introduction-to-langgraph-a-step-by-step-tutorial-2b314c967d3c)

---

## 15. Hands-On Exercises

### Exercise 1: Build Your First Graph

**Task:** Create a simple graph that:

1. Takes a customer email as input.
2. Classifies it as "urgent" or "routine".
3. Routes to different response nodes.
4. Combines final response.

**Hints:**

- Use 4 nodes: `classify`, `urgent_response`, `routine_response`, `format_output`.
- Use conditional edge after `classify`.
- Test with sample emails.

---

### Exercise 2: Implement State Appending

**Task:** Modify the graph to:

1. Keep track of all processing steps.
2. Store intermediate results.
3. Return complete history.

**Requirements:**

- Use `Annotated[list, operator.add]`.
- Each node should append to history.
- Final output shows full processing chain.

---

### Exercise 3: Create Custom Operator

**Task:** Build a custom operator that:

1. Merges nested dictionaries intelligently.
2. Handles conflicts by keeping most recent value.
3. Maintains metadata timestamps.

```python
def smart_merge(dict1, dict2):
    # Your implementation here
    pass
```

---

### Exercise 4: Parallel Processing

**Task:** Create a graph that processes data in parallel:

1. Extract content from input.
2. Simultaneously: (a) Analyze sentiment, (b) Extract keywords, (c) Detect language.
3. Combine all results in final node.

**Bonus:** Add timing information to see parallel speedup.

---

## 16. Debugging Tips

### Common Issues

#### 1. State variables not updating

- Check that you’re returning a dictionary from functions.
- Verify variable names match the State class.

#### 2. Conditional edge errors

- Ensure routing function returns an **exact key** from the mapping.
- Check for typos in node names.

#### 3. Graph visualization shows unexpected flow

- Review edge definitions.
- Verify conditional logic.

#### 4. Import errors

- Install: `pip install langgraph langchain`
- Check Python version (3.9+).

---

## 17. Complete Code Reference

```python
from typing_extensions import TypedDict
from typing import Annotated
import operator
from langgraph.graph import StateGraph, END, START

# Define State
class State(TypedDict):
    text: str
    answer: Annotated[list, operator.add]
    payload: dict[str, list]

# Define node functions
def extract_content(state: State):
    return {"text": state["payload"][0]["customer_remark"]}

def route_question_or_compliment(state: State):
    if "?" in state["text"]:
        return "question"
    else:
        return "compliment"

def run_compliment_code(state: State):
    return {"answer": ["Thanks for the compliment."]}

def run_question_code(state: State):
    return {"answer": ["Wow nice question."]}

def beautify(state: State):
    return {"answer": [state["answer"][-1] + " beautified"]}

# Build graph
graph_builder = StateGraph(State)

# Add nodes
graph_builder.add_node("extract_content", extract_content)
graph_builder.add_node("run_question_code", run_question_code)
graph_builder.add_node("run_compliment_code", run_compliment_code)
graph_builder.add_node("beautify", beautify)

# Add edges
graph_builder.add_edge(START, "extract_content")
graph_builder.add_conditional_edges(
    "extract_content",
    route_question_or_compliment,
    {"compliment": "run_compliment_code", "question": "run_question_code"},
)
graph_builder.add_edge("run_question_code", "beautify")
graph_builder.add_edge("run_compliment_code", "beautify")
graph_builder.add_edge("beautify", END)

# Compile
graph = graph_builder.compile()
```

---

## 18. References

- **Gentle Introduction to LangGraph: A Step-by-Step Tutorial** (Dr. Varshita Sher)  
  [https://levelup.gitconnected.com/gentle-introduction-to-langgraph-a-step-by-step-tutorial-2b314c967d3c](https://levelup.gitconnected.com/gentle-introduction-to-langgraph-a-step-by-step-tutorial-2b314c967d3c)
- Cloud LLM used for slides generation.
