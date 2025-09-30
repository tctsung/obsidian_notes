---
created: 2025-09-01T19:38
updated: 2025-09-28T21:31
---


## Intro
- synergizing <span style="color:rgb(255, 0, 0)">re</span>asoning & <span style="color:rgb(255, 0, 0)">act</span>ion
- A prompt framework to let LLM:
	- able to use **external resources** (the <span style="color:rgb(255, 0, 0)">tools</span>)
	- iteratively do **complex reasoning**
- requirements:
	- large scale models
- benchmarks
	- HotPot QA: multi-step Q&A
	- Fever: fact verification
- code application note: [[LLM/framework/LangGraph/ReACT|ReACT]]
### pipeline

> **Question → thought → action → observation**
- **Q**: understand the question 
    - eg. which magzine started first? Arthur’s or First of Women
- **T**: reasoning step, demonstrate how to tackle the problem & take action
    - eg. I need to search for Arthur’s & First of Women, then find which one started first
- **A**: action from allowed set of tools
    - eg. trigger search tool/API to search (Arthur’s magazine)
- observation:
    - result you got from action
repeats the above cycle until get the final answer
### tool use
1. LLM has context on of how to use the tool & what can the tools do
2. LLM take user input & if decide to use tool 
	$\rightarrow$ return structured output of tool name & input arguments
3. executes tool, returns result back to LLM
4. LLM continues reasoning with new info (next iteration)
