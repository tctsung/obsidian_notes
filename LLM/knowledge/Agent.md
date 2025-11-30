---
created: 2025-08-03T22:05
updated: 2025-09-28T21:15
---
# Intro

## Chain vs agents

- **Chain** always have the same workflow
    - fixed control flow
    - reliable, good for task eg. info extraction
- **Agents** is more flexible, can pick their own workflow
    - LLM-defined control flow

![Chain](image.png)

Chain

![Agent](image%201.png)

Agent

## Types of agents

- **Router**: let LLM control a single node of flow
- **Fully autonomous**
    - choose step from a bunch of options
    - generate steps by itself

![image.png](image%202.png)  ![image.png](image%203.png)

- In practice, more agent’s lvl of control → **reliability** decrease
    - tooo flexible
    - This is why LangGraph aims to improve:
        - **balance reliability with LLM control**
    
    ![image.png](image%204.png)