---
created: 2025-09-01T19:38
updated: 2025-09-01T19:40
---
* breakthrough: link LLM with external sources (eg. function/)
	* framework for 
	* synergizing reasoning & action

- a 
    
    - can do complex reasoning
    - bridges the gap between reasoning and acting
    - give LLM better language understanding & decision making
- usually need large scale models
    
- application:
    
    - multi-step Q&A
- banchmarks:
    
    - HotPot QA: multi-step Q&A
    - Fever: fact verification
- use structured examples as shot
    
    - the model should **follow** this pipeline
    
    **Question → thought → action → observation**
    
    - Q: a problem that need advanced reasoning & multiple steps to solve
        - Q. which magzine started first? Arthur’s or First of Women
    - T: reasoning step, demonstrate how to tackle the problem & take action
        - I need to search for Arthur’s & First of Women, then find which one started first
    - A: **external** task from allowed set of actions
        - generate python code to trigger API
        - search [entity] → lookup [string] → finish [answer]
        - eg. search [Arthur’s magazine]
    - observation:
        - result you got from action
    - repeats the cycle until get the final answer
        - Cycle 2:
            
            ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/1c8cb682-5337-4672-b0d9-ec868f326a34/e91ebeae-c43b-4634-a089-f7d2104c574a/Untitled.png)
            
        - Cycle 3:
            
            ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/1c8cb682-5337-4672-b0d9-ec868f326a34/fd211adf-5a59-436e-b148-ebf087ef4842/Untitled.png)
            

## ReAct action space

- Solve a Q&A task **with interleaving thought, action, observation steps**
    - give LLM a workflow to follow
- Give more details about what is meant by thought, and define the actions LLM can choose
    - **limit options for LLM to choose, $\because$ LLM is too creative**
    - Actions can be below 3 steps:
        1. search [entity]: search the exact/similar entity on wiki & returns paragraph
        2. lookup[keyword]: return the next sentence containing keyword in current passage
        3. finish[answer]: return answer & finish the task