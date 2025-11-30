---
created: 2025-09-28T19:34
updated: 2025-11-30T00:39
---
## Intro
- ref
	- [langgraph pre-build agent](https://langchain-ai.github.io/langgraph/agents/agents/)
- A prompt framework to let LLM:
	- able to use **external resources** (the <span style="color:rgb(255, 0, 0)">tools</span>)
	- iteratively do **complex reasoning**
- detailed knowledge note: [[ReAct Knowledge|ReAct Concept]]


## Code

### prep
- prepare input args for react agents
- `State`
	- always inherit from <span style="color:rgb(255, 0, 0)">AgentState</span>!!! Otherwise might not be able to use `Command()` to update State directly in tools
	- state `message` will be passed to ReACT agent automatically
		- eg. ToolMessage/string returned from tools
```python
from langchain.chat_models import init_chat_model
from langchain_core.tools import tool

## base model
my_llm = init_chat_model(...)

## list of python functions, ideally decorated by langgraph @tool
@tool
def web_search(...)
	...
react_tools = [web_search, retreiver, check_gibberish]

## State (always inherit from AgentState!!!!!!!!!)
from langgraph.prebuilt.chat_agent_executor import AgentState

class MyState(AgentState):
	xxx: str
```

### Create agent
```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
	model = my_llm,
	tools = react_tools,
	prompt = "Role: You're a Amazon Ads expert with 27 years of experience",
	response_format = MyState
)
```

### Invoke/generate response
- Invoke
```python
init_state = dict(
	state_arg1 = ...
	state_arg2 = ...
	)

response = agent.invoke(init_state)

# get final state info (as a dictionary)
response.get("state_arg1", "")  
```
- streaming (better to record the intermediate process)
	- recommend to use `stream_mode = "values"`: to get the full state
		$\rightarrow$ then use `['messages']` to log ongoing agent context
```python
for cur_state in agent.stream(
	init_state,
    stream_mode="values",
	):
	current_msg = cur_state['messages'][-1]['content']  # latest msg
	logging.info(current_msg)   # log current message
final_state = cur_state         # last iter chunk contains the final state

# get final state info (as a dictionary)
final_state.get("state_arg1", "")  
```

## Prompt example
```
Answer the following questions as best you can. 
You have access to the following tools:

{tools}

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought:{agent_scratchpad}
```