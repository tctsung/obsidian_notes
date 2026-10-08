---
created: 2026-10-07T23:50
updated: 2026-10-08T00:10
---
## Introduction
- def: single-thread concurrency, <span style="color:rgb(255, 0, 0)">switch to other tasks while one waits</span> on I/O
- motivation: programs often get **stuck waiting** on I/O (API request, DB query, file read)
	- CPU sits idle during the wait -> 10 API calls × 2s = 20s, with almost 0s of real work
- how: set a switch point (`await`) at every wait -> run other tasks while waiting
	- ⭐ <span style="color:rgb(255, 0, 0)">speedup comes from other tasks running during the wait</span>
	- `await` itself is NOT faster: 1 API call takes the same time in sync & async
- other concurrency methods (not covered here)
	- multithreading
	- multiprocessing

### Key concepts
| term        | def                                                                       | note                                                                                                                         |
| ----------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| coroutine   | function defined with `async def`, can **pause in the middle**            | calling it does NOT run it, only creates a coroutine object -> runs when `await`ed or wrapped in a task                       |
| `await`     | "wait here until the result arrives", let other tasks run meanwhile       | code after `await` runs only once the result is ready; switch only when it <span style="color:rgb(255, 0, 0)">really has to wait</span> (eg. network) |
| task        | a coroutine handed to the event loop (`create_task`, `gather`)            | multiple tasks = the only source of speedup                                                                                  |
| event loop  | the scheduler that runs tasks **one at a time**, switching at every wait  | waiting tasks are parked; OS notifies the loop when a response arrives -> task is ready again (no polling, CPU is free)         |
| concurrency | tasks overlap in time by **taking turns**                                 | async, threads                                                                                                               |
| parallelism | tasks run **at the same instant** on diff cores                           | multiprocessing, GPU; async = concurrency only                                                                               |

- event loop step by step (2 agents, each does `await llm()` then `await search()`)
```
ready: [q1, q2]
run q1 -> hits await llm()     -> q1 parked (waiting for network)
run q2 -> hits await llm()     -> q2 parked
nothing ready -> loop idles until OS: "q1's response arrived"
run q1 -> hits await search()  -> q1 parked
run q2 -> hits await search()  -> q2 parked
... -> both finish in ~2 waits instead of 4
```

## Q&A
- Q: what should be `await`ed?
	- Any call that waits on the outside world, such as an HTTP request, a DB query or an LLM call. It must come from an async library (`httpx`, `asyncpg`, `AsyncAnthropic`), because only those return something you can `await`. Plain computation like JSON parsing runs directly.
	- only a blocking library available (`requests`, `boto3`)? wrap it: `await asyncio.to_thread(requests.get, url)`
- Q: is async just threads with more control?
	- Mostly, yes. The OS can pause a thread at any line, but a coroutine pauses only where you write `await`. The catch is that async runs on <span style="color:rgb(255, 0, 0)">one thread</span>, so a task that never awaits (a heavy loop, `time.sleep`) freezes every other task.
	- vs threads for I/O: a coroutine costs a few KB and a thread costs MBs of stack, so async handles thousands of connections easily. Threads still win when the library is blocking-only.
- Q: what if the next line needs the result?
	- Then it waits, which is what you want. Await steps one by one when each needs the previous result, and `gather` the ones that don't (see [[#Mix dependent & independent steps]]).
- Q: must we define all concurrent tasks at the start?
	- No. `gather` suits a fixed set, but `create_task` can start a new task at any point. A web server works this way: each incoming request becomes a new task.
- Q: why do LLM SDKs offer both sync & async clients?
	- The sync client is simpler, and just as fast when you make one call at a time, as in a script or notebook. The async client pays off when many calls wait at once: 100 summaries finish in about the time of the slowest one, and a server keeps answering other users while one request waits on the LLM.

## Practical cases
### 1. Basic async & await
- each agent has 2 `await`s (= 2 switch points)
	- within one agent: steps run **in order**
	- across agents: steps **interleave** (while q1 waits on search, q2 makes progress)
```python
import asyncio, time

async def llm(prompt):
    await asyncio.sleep(1)                  # stand-in for an LLM API call
    return f"plan for {prompt}"

async def search(query):
    await asyncio.sleep(1)                  # stand-in for a web search API
    return f"docs for {query}"

async def agent(question):
    print(f"{question}: start")             # runs right away when the task gets its turn
    plan = await llm(question)              # switch point 1
    docs = await search(plan)               # switch point 2
    print(f"{question}: done")
    return docs

async def main():
    t = time.time()
    await agent("q1"); await agent("q2")              # sequential
    print(f"{time.time() - t:.1f}s")                  # > 4.0s

    t = time.time()
    await asyncio.gather(agent("q1"), agent("q2"))    # concurrent
    print(f"{time.time() - t:.1f}s")                  # > 2.0s

asyncio.run(main())
# > q1: start, q1: done, q2: start, q2: done, 4.0s
# > q1: start, q2: start, q1: done, q2: done, 2.0s
```
- real code: replace `asyncio.sleep` with an async client call
```python
from anthropic import AsyncAnthropic
client = AsyncAnthropic()

async def llm(prompt):
    msg = await client.messages.create(
        model="claude-sonnet-5-5", max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )
    return msg.content[0].text
```
- common bugs
```python
r = client.get(url)    # forgot await -> coroutine object, not result ("never awaited" warning)
time.sleep(1)          # inside async def -> blocks the whole event loop, use await asyncio.sleep(1)
```

### 2. Background task
- `create_task`: schedule a task in background, keep running other code, `await` the result when needed
```python
async def fetch_data():
    await asyncio.sleep(2)                    # stand-in for an API call
    return "data"

async def main():
    task = asyncio.create_task(fetch_data())  # scheduled, returns immediately
    print("doing other work...")
    await asyncio.sleep(1)                    # other async work -> background task gets its turn
    print(task.done())                        # > False (needs 1 more sec)
    result = await task                       # only waits for the remaining time
    print(result, task.done())                # > data True

asyncio.run(main())
```
- rarely need to poll `task.done()` -> just `await task`: returns instantly if done, else waits for the remaining time
- background task <span style="color:rgb(255, 0, 0)">only runs while main code is paused at an await that really waits</span> -> `heavy_cpu_loop()` without await = task never starts
	- CPU-heavy work -> `asyncio.to_thread` keeps the loop responsive (no speedup, GIL); true parallelism -> `ProcessPoolExecutor`
- keep a reference (`task = ...`): event loop only holds a weak ref -> unreferenced task can be garbage-collected mid-run

### 3. Other common tools
| tool                          | use when                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| `asyncio.gather(a(), b())`    | fixed set of tasks, want all results together                |
| `asyncio.create_task(a())`    | start in background, `await` the result later                |
| `asyncio.Semaphore(n)`        | limit num of concurrent tasks (eg. API rate limit)           |
| `asyncio.to_thread(f, *args)` | run a blocking sync function without blocking the event loop |

#### Mix dependent & independent steps
```python
async def handle(user_id):
    user = await get_user(user_id)              # needed by next lines -> await first

    orders, profile = await asyncio.gather(     # both need user, but not each other -> run together
        get_orders(user),
        get_profile(user),
    )
    return summarize(user, orders, profile)     # CPU work, no await
```
#### Batch LLM calls with rate limit
- `Semaphore(n)`: at most n tasks inside `async with sem` at once, the rest wait
```python
sem = asyncio.Semaphore(5)                      # max 5 requests in flight

async def summarize(doc):
    async with sem:
        return await llm(f"summarize: {doc}")

async def main(docs):
    return await asyncio.gather(*(summarize(d) for d in docs))
```

## PyTorch GPU timing
- GPU ops are <span style="color:rgb(255, 0, 0)">async to the CPU by default</span>: `y = x @ x` only **queues a kernel** (GPU job) & returns immediately
	- GPU is a separate processor -> keeps working while Python moves on (true parallelism)
- `torch.cuda.synchronize()`: CPU waits until all queued GPU work is done -> timing without it = launch time, NOT compute time
```python
x = torch.randn(8192, 8192, device="cuda")
x @ x; torch.cuda.synchronize()               # warmup (1st matmul includes one-time setup) & empty the queue

t = time.perf_counter()
y = x @ x
torch.cuda.synchronize()                      # wait for GPU to finish
print(f"{(time.perf_counter() - t) * 1000:.2f} ms")   # > real compute time (without sync: tiny)
```
- hidden sync points (need value on CPU): `.item()`, `.cpu()`, `print(tensor)` -> avoid `loss.item()` every step
- Mac (MPS): `torch.mps.synchronize()`
