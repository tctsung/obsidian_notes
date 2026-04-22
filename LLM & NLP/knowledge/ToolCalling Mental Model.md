---
created: 2025-12-27T19:48
updated: 2025-12-27T22:33
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
	- conversation history
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
	- conversation history
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
```python
import json
sys_prompt = """
ROLE: You are a helpful assistant

TOOLS:
1. count_r(query: str) -> int
   Description: Tell a joke in Deron style
   Parameters: query (input text to count number of `r`)
2. add_numbers(a: int, b: int) -> int
   Description: Adds two numbers.
   Parameters: a (First number), b (Second number)
When tool call needed, output ONLY a nested xml with tags <tool_call>


Output Example:
<tool_call> 
	<name>add_numbers</name>
	<args>{"a": 5, "b": 3}</args>
</tool_call> 
"""

user_prompt = "How many r in strawberry?"

# tools:
def count_r(query: str) -> int:
  return query.count('r')
def add_numbers(a: int, b: int) -> int:
  return a + b
tools = {"count_r": count_r, "add_numbers": add_numbers}

# parse toolcall output:
def run_tool(txt, tools):
    try:
        # Logic: Split by the opening tag, take the right side. 
        # Then split by closing tag, take the left side.
        name = txt.split("<name>")[1].split("</name>")[0]
        args = txt.split("<args>")[1].split("</args>")[0]
        
        return tools[name.strip()](**json.loads(args))
    except Exception as e:
        return f"Error: {e}"


# LLM call:
response = llm.invoke([SystemMessage(content=sys_prompt), HumanMessage(content="How many r in strawberry?")])
llm_response = response.content[0]['text']
print(f"LLM response:\n {llm_response}")
print("=================")
toolcall_output = run_tool(llm_response, tools)
print(f"Tool Output:\n {toolcall_output}")
```


- result:
```

LLM response:
 <tool_call>
	<name>count_r</name>
	<args>{"query": "strawberry"}</args>
</tool_call>
=================
Tool Output:
 3
```