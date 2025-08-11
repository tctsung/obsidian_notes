---
created: 2025-08-03T15:08
updated: 2025-08-10T23:25
tags:
  - tavily
  - code
description: 
---

---


## Tavily Intro
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

```python
import json
from langchain_core.messages import ToolMessage

class BasicToolNode:
    """A node that runs the tools requested in the last AIMessage."""

    def __init__(self, tools: list) -> None:
        self.tools_by_name = {tool.name: tool for tool in tools}

    def __call__(self, inputs: dict):
        if messages := inputs.get("messages", []):
            message = messages[-1]
        else:
            raise ValueError("No message found in input")
        outputs = []
        for tool_call in message.tool_calls:
            tool_result = self.tools_by_name[tool_call["name"]].invoke(
                tool_call["args"]
            )
            outputs.append(
                ToolMessage(
                    content=json.dumps(tool_result),
                    name=tool_call["name"],
                    tool_call_id=tool_call["id"],
                )
            )
        return {"messages": outputs}


tool_node = BasicToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)
```


