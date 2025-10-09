---
created: 2025-08-03T13:10
updated: 2025-10-08T22:46
tags: []
description:
---


## Intro
* def: wrapper to specify msg type in organized way & have extra metadata

## Chat messages
* basic form: dict
* useful for chat models, avoid using Base message, can't invoke

```python
# basic form:
a_human_msg = {"content": "Tell me a joke", "type": "user"}



from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
output = llm.invoke([
				SystemMessage("You are a angry chatbot"), 
				HumanMessage(content="What's good") 
			])
```
## Tool Calling
* See [[Tool calling]] 
* create a tool calling node

```python
from langchain_core.messages import ToolMessage
ToolMessage(
		content=json.dumps(tool_result),
		name=tool_call["name"],
		tool_call_id=tool_call["id"],
        )
```