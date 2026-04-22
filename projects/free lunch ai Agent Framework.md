---
created: 2025-12-17T23:40
updated: 2025-12-28T14:11
---


### Speciality
- integrate multiple models automatically (like LiteLLM)
- only include Free ones (FreeAgent, Free)
	- load_api_keys()  
- automatically choose the proper model based on usecase (router?)
- automatically have a RANK for different usecase (eg. speed, reasoning, agent)
### Goal
- turn into a python package that I will use everyday for my projects (the PCT project, RoleLLM)
- hand-write the commonly used functions (eg. bind_tool, RAG, react, structured output, stream)
	- use `httpx` to have a lightweight version (??? might not worth it???)
- have langgraph version of things (for agent integration)
### Demo usecase
- dependencies
	- stick to langchain-core & langgraph if possible
	- might need others for agent, but NOT for self-written one

```python
# pip install freelunch[langgraph]
# pip install freelunch
import freelunch as free
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage

# load the keys into sys env
free.load_api_keys("~/project_free/.env")

# simple interface (useful to be individual tool)
bot = free.chatbot(model="auto", goal=Literal["fast", "reasoning"])
# > auto = automatically choose model & swith when reach api limit

bot.invoke("what's good!")
# > return: str

# interface for langgraph agent workflow
model = free.langchain_llm(model="<provider>:<model id>")
model.invoke([SystemMessage("..."), HumanMessage("...")])
agent = lang


# self-written practice of an agent
free.agent(
	model="<provider>:<model id>",  # or model name only
	tools=[func1, func2],
	structured_output = dataclasses/typeddict/pydantic
	)
# > can practice toolcall, structured output here
```