---
created: 2025-08-03T15:08
updated: 2025-09-01T23:34
tags:
  - tavily
  - code
description:
---

## Tool Calling
* def: Let LLM able to use external tools/API outside of 文字接龍
	* key for ReACT: tool calling & reasoning loop
	* each tool calls cost an API call/LLM inference
* basic ReACT:
	1. give LLM instruction of how to use the tool & what can the tool do
	2. LLM take user input & if decide to use tool 
		$\rightarrow$ return structured output of tool name & input arguments
	3. executes tool, returns result back to LLM
	4. LLM continues reasoning with new info
### Tavily (Websearch tool)
* API call: do websearch & return webpage summaries of top k results with Relevance Scoring
	* might use some NLP models in backend for summary/re-ranking
* pre-requisite: load **TAVILY_API_KEY** as environment var
```python
from langchain_tavily import TavilySearch
tool = TavilySearch(max_results=2)
tool.invoke("How does Tavily Search work?")
```

* output example
![[Pasted image 20250810160757.png]]

## Code Example
### Tool prep
*  **`@tool`** decorator: **auto-generate metadata** from python func 
	must make the following NLP readable:
	* Function name: tool’s name
	* docstring: digest tool description, arguments, return values
		* use <span style="color:rgb(255, 0, 0)">"Parameters:"</span> or <span style="color:rgb(255, 0, 0)">"Args:"</span> for params
		* don't use "TODO", start with description directly
		* use <span style="color:rgb(255, 0, 0)">"Returns:"</span> for return description
	* schema: use <span style="color:rgb(255, 0, 0)">typehints</span>, eg. **`-> str`** & **`: int`**
* **`llm.bind_tools`**: feed tool name, args, etc. to LLM context window
```python
from langchain.chat_models import init_chat_model

# LangChain-compatible tool object
from langchain_tavily import TavilySearch
tavily_search = TavilySearch(max_results=2)

# self-written function:
@tool
def deron_style_joke(n: int = 2) -> str:
    """
    Tell n short jokes in Deron's style (dark humor)

    Parameters:
        n (int, optional): Number of jokes to return. Defaults to 2

    Returns:
        str: A string containing `n` jokes, separated by newlines..
    """
    jokes = [
		"I have a joke about procrastination… but I’ll tell you later.",
		"I broke my finger last week. On the other hand, I’m okay.",
		"Why don’t cannibals eat clowns? They taste funny.",
		"I used to play piano by ear. Now I use my hands.",
    ]
    return "\n".join(jokes[:n])

## bind tools to LLM (will add to context window automatically)
tools = [tavily_search, deron_style_joke]
llm = init_chat_model("google_genai:gemini-2.5-flash")
llm_with_tools = llm.bind_tools(tools)
```
### Build Graph
 * **`ToolNode`**: turn a list of python function (the tools) into a node available for LLM
 *  **`tools_condition`**: conditional edge that checks **did the LLM try to call a tool**
	* if YES: route to tool node (must name the tool node <span style="color:rgb(255, 0, 0)">"tools"</span>)
	* if NO: route to END (finish the ReACT loop)
	![[Pasted image 20250901232319.png]]
```python
from langgraph.prebuilt import ToolNode, tools_condition
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from typing import Annotated

# State:
class MsgState(TypedDict):
    messages: Annotated[list, add_messages]
# Node 1. LLM:
def chatbot(state: MsgState):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}
# Node 2. tool:
tool_node = ToolNode(tools=tools)

# build graph:
graph_builder = StateGraph(MsgState)
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", tool_node) # must call it tools

graph_builder.add_edge(START, "chatbot")
graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,  
) # -> return to "tools" or go to END
graph_builder.add_edge("tools", "chatbot") # back to chatbot to decide next step
graph = graph_builder.compile()
```
### Execution
```python
## Ex. 1
llm_output = graph.invoke({
	"messages":
	"Go search online, What date is today? And what timezone in New York in"})
llm_output["messages"][-1].content
# > Today's date is Monday, September 1, 2025. New York is in Eastern Time Zone, 
# > which is currently Eastern Daylight Time (EDT), with an offset of GMT-4.

## Ex. 2
llm_output = graph.invoke({"messages":"Tell me 1 Deron style joke"})
llm_output["messages"][-1].content
# > I have a joke about procrastination… but I’ll tell you later.
```
* Output:
![[Pasted image 20250901233029.png]] | ![[Pasted image 20250901233334.png]]