---
created: 2025-08-03T15:08
updated: 2025-10-08T23:26
tags:
  - tavily
  - code
description:
---

## Introduction
* def: Let LLM able to use external tools/API outside of 文字接龍
	* key for ReACT: tool calling & reasoning loop
	* each tool calls cost an API call/LLM inference
## Mental Model
*** note: I use XML tags for demo, but usually it's json format
1. In LLM context window, provide a list of tool & instruction on how to output structured tool calls (json-like format)
	- code:
	```python
	llm = init_chat_model("google_genai:gemini-2.5-flash")
	llm_with_tools = llm.bind_tools(tools)
	```
	- behind the hood: simply adding tool call instruction into system prompt
```
<|system|>
You are a helpful assistant.
------prompt added from bind_tools--------------------
TOOLS:
1. TavilySearch(query: str) -> int
   Description: Do websearch with Tavily
   Parameters: query for websearch
2. add_numbers(a: int, b: int) -> int
   Description: Adds two numbers.
   Parameters: a (First number), b (Second number)
When needed, output ONLY a xml with tags <tool_call>
<tool_call> 
	<name>add_numbers</name>
	<args>{"a": 5, "b": 3}</args>
</tool_call> 
------end of prompt added from bind_tools-------------
```

2. user input $\rightarrow$ if LLM think needs a tool, will return structured output calls
	- mock code:
	```python
	conversation_history = []
	user_input = HumanMessage(content="Add 5 and 3")
	conversation_history.append(user_input)
	llm_output = llm_with_tools.invoke(messages)
	
	# Inspect if LLM output has tool_call XML tag:
	if "<tool_call>" in llm_output:
		# parse it to handy data str (eg. dict)
		tool_call_info = parse_tool_call_tag(llm_output)
		# > {"name": "add_numbers", "args": {"a": 5, "b": 3}, "id": "uuid"}
	```
	- LangGraph automation
		- append a `AIMessage` with tool call info into state `messages`
```python
conversation_history.append(AIMessage(tool_call_info))
```
3. Use the parsed tool name & input arg to run associated function
	- mock code
	```python
	available_tools = {"add_numbers": add_numbers, "TavilySearch": TavilySearch}
	if tool_call_info.get("name") in available_tools:
		tool_func = available_tools[tool_call_info.get("name")]
		tool_output = tool_func(**tool_call_info.get("args"))
	```
	- LangGraph automation
		- wrap tool output into `ToolMessage` & appended to conversation history
```python
conversation_history.append(
	ToolMessage(
		content=tool_output, 
		tool_call_id=tool_call_info.get("id")
		)
	)
```
#### Example
/!!!!!!!! add a screenshot of LangGraph stream/


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
	 * automatically return the `{"messages": "msg to toolcalling LLM"}`
	 * 
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