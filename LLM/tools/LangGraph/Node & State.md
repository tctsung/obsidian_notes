---
created: 2025-08-03T13:10
updated: 2025-08-10T16:21
tags:
  - code
  - LangGraph
description: Node/State/Edge
---


- [ref Colab notebook](https://colab.research.google.com/github/langchain-ai/langchain-academy/blob/main/module-1/simple-graph.ipynb)

## Main Components

| Name       | def                                                       | notes                                                                                                                                 |
| ---------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| State      | **Schema** of node/edge                                   | store & pass info btw nodes                                                                                                           |
| Node       | basic **unit** of agentic workflow, a **python function** | 1. 1st arg must be **State**<br>2. output an update to the state                                                                      |
| Edge       | to **connect** the nodes with logic                       | - 1st arg must be **State**<br>- **normal edge**: fixed workflow<br>- **conditional edge**: LLM/schema help decide workflow direction |
| StateGraph | workflow: stores all nodes & edges                        |                                                                                                                                       |


### State & Schema
- the graph_state will be overwrite for each node
- use TypedDict for protyping, BaseModel for production
	- less dependencies
- `Annotated`: 
	- 1st arg: declared attribute datatype
	- 2nd arg: metadata, in langChain, this will be the **reducer func** (the func for updating the attribute)
		- use this well to customized special updates (eg. add_messages is a built-in reducer)

```python
from typing_extensions import TypedDict
from typing import Annotated
from pydantic import BaseModel

# typed dict:
class TypeState(TypedDict):
    graph_state: str

# pydantic:
class PydaState(BaseModel):
    graph_state: str

# eg. get attr:
state = TypeState()
state["graph_state"]

state = PydaState()
state.graph_state
```

### Node

- python function with 1st arg being State
- should <span style="font-weight:bold; color:rgb(255, 0, 0)">ALWAYS return a dict</span>
    - $\because$ this is how LangGraph update the state attr
- if key in return dict can be found in state schema → update state
    - will always override prior state value
- if key does NOT exist in state schema→ ignore the key (unless use default `MessagesState`)

```python
# Syntax
def node(state):
    return {<attr in state schema>: <new value>}

# Example:
def node_1(state: TypeState):
    new_state = state['graph_state'] +" I am"
    return {"graph_state": new_state}   # overwrite prev state

def node_2(state: PydaState):
    return {"graph_state": state['graph_state'] +" happy!"}
def node_3(state: PydaState):
    return {"graph_state": state['graph_state'] +" sad!"}
```

### Edge

- def: python function to connect nodes, 1st arg always a State
- **Normal Edge**: fixed workflow (eg. always node A → B)
- **Conditional Edge**: use schema/LLM to help decide workflow (node A → node B/C/D)

```python
from typing import Literal   # restrict output from a set of categories
def decide_mood(state) -> **Literal**["node_2", "node_3"]:  
    
    # Often, we will use state to decide on the next node to visit
    user_input = state['graph_state'] 
    
    # Here, let's just do a 50 / 50 split between nodes 2, 3
    if random.random() < 0.5:

        # 50% of the time, we return Node 2
        return "node_2"
    
    # 50% of the time, we return Node 3
    return "node_3"
```

## Graph Construction

- LangGraph runs **<span style="color:rgb(255, 0, 0)">synchronously</span>** (wait for previous node to complete before starting the next node)****
- `StateGraph` stores all nodes & edges and compile a workflow
    - 1st arg is **central state** → a state that includes all attr in all nodes
    - central state is updated by the nodes over workflow
    - have to define even if each node has state $\because$ `state` is at Graph lvl, not node lvl
- `START` and `END` are two special nodes for start & end of workflow

```python
from langgraph.graph import StateGraph, START, END

# Build graph
builder = StateGraph(State)  # init workflow obj

# add all required nodes to graph
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)

# add edge to define logic btw nodes
builder.add_edge(START, "node_1")
builder.add_conditional_edges("node_1", decide_mood)  # decide_mood -> node_2 | node_3
builder.add_edge("node_2", END)
builder.add_edge("node_3", END)

# compile the graph:
graph = builder.compile()
```

### visualization

- built-in has Mermaid Diagram attr

```python
from IPython.display import Image
display(Image(graph.get_graph().draw_mermaid_png()))
```

![image.png](assets/image%205.png)

### Invoke / initialize the graph

`graph.invoke({<state attr>: <initial value>})`

```python
graph.invoke({"graph_state" : "Hi, this is Lance."})
# > if node_2: Hi, this is Lance. I am happy
```

## ChatBot

### State with reducers

- TODO: **append** the state attr instead of overwriting it

```python
from typing import Annotated
from langchain_core.messages import AnyMessage
from langgraph.graph.message import **add_messages**

# hand-write:
class MessagesState(TypedDict):
    **messages**: Annotated[list[AnyMessage], add_messages]
    
# LangGraph have built-in to append list:
**from langgraph.graph import MessagesState**  # default has **messages** that append msg
**class MessagesState(MessagesState):
    pass**
    
# Nodes & edges
def llm_msg(state: **MessagesState**):
    return {"messages": [llm.invoke(state["messages"])]}
```

- Self-written replacement
    - MessagesState has an issue: only work with LangChain models

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
def append(prev: list, msg):
    if not isinstance(msg, list):
        msg = list(msg)
    output = prev + msg
    return output
class **MsgState**(TypedDict):
    messages: Annotated[list[dict], append]

def llm_msg(state: MsgState):
    llm_output = llm.run(messages=state["messages"])
    print("AI:\n", llm_output)
    AI_response = {"role": 'assistant', "content": llm_output}
    return {"messages": AI_response}

def user_msg(state: MsgState):
    user_input = input("type 'end' to end conversation: ")
    print("User:\n", user_input)
    user_msg = {"role": 'user', "content": user_input}
    return {"messages": user_msg}

def end_or_continue(state: MsgState):
    if state["messages"][-1]["content"] == "end":   # if latest user message is "end"
        return END
    else:
        return "llm_msg"
# Build graph

```

### Graph

- Node & edges

```python
# build graph:
builder = StateGraph(MsgState) 
builder.add_node("llm_msg", llm_msg)
builder.add_node("user_msg", user_msg)
builder.add_edge(START, "user_msg")
builder.add_edge("llm_msg", "user_msg")
builder.add_conditional_edges("user_msg", end_or_continue)
graph = builder.compile()

# start conversation:
graph.invoke({"messages": []})
```

![image.png](assets/image%206.png)