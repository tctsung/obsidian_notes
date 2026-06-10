---
created: 2026-06-09T23:52
updated: 2026-06-10T10:56
---

I built [free-lunch-ai](https://github.com/tctsung/free-lunch-ai) as a reusable module for my own projects — **single entry point for any free LLM provider, plus reusable built-in tools**. Set one env key per provider and go.
```bash
pip install "free-lunch-ai[langchain] @ git+https://github.com/tctsung/free-lunch-ai.git"
```

## Quick Start
> Source: [llm_factory.py](https://github.com/tctsung/free-lunch-ai/blob/main/src/free_lunch/llm_factory.py)

```python
from free_lunch import LangChainFactory

llm = LangChainFactory.create(
    "groq::llama-3.3-70b-versatile",
    temperature=0,
    max_tokens=8192,
)
```
Returns `BaseChatModel` — compatible with all LangChain / LangGraph APIs.
- For failover routing across multiple models, see [repo examples](https://github.com/tctsung/free-lunch-ai/blob/main/examples/demo.ipynb).

---
## Groq
> ref:  [API key](https://console.groq.com/keys) | [Rate limits](https://console.groq.com/docs/rate-limits) | [Reasoning docs](https://console.groq.com/docs/reasoning)

```python
# bash: export GROQ_API_KEY=...
llm = LangChainFactory.create(
    "groq::llama-3.3-70b-versatile",
    temperature=0,
    max_tokens=8192,
)
llm = LangChainFactory.create(
    "groq::openai/gpt-oss-120b",
    reasoning_effort="high",
)
llm = LangChainFactory.create(
    "groq::qwen/qwen3-32b",
    reasoning_effort="default",  # enable thinking
)
```
### Reasoning config
| Param | Models | Values |
|-------|--------|--------|
| `reasoning_effort` | gpt-oss-20b, gpt-oss-120b | `"low"`, `"medium"`, `"high"` |
| `reasoning_effort` | qwen/qwen3-32b | `"none"` (disable), `"default"` (enable) |
| `reasoning_format` | qwen/qwen3-32b | `"parsed"`, `"raw"`, `"hidden"` |

---
## Google AI Studio
> ref: [aistudio.google.com/apikey](https://aistudio.google.com/apikey) | [Models](https://ai.google.dev/gemini-api/docs/models) | [Pricing/limits](https://ai.google.dev/pricing)

```python
# bash: export GOOGLE_API_KEY=...
llm = LangChainFactory.create(
    "google::gemini-2.5-flash",
    temperature=0.1,
    max_tokens=4096,
)
llm = LangChainFactory.create(
    "google::gemini-2.5-flash",
    thinking_budget=1024,      # Gemini 2.5
)
llm = LangChainFactory.create(
    "google::gemini-3.5-flash",
    thinking_level="low",      # Gemini 3+
)
```

### Thinking config
| Param             | Models     | Values                                        |
| ----------------- | ---------- | --------------------------------------------- |
| `thinking_budget` | Gemini 2.5 | `0` (off), `-1` (dynamic), or int (token cap) |
| `thinking_level`  | Gemini 3+  | `"minimal"`, `"low"`, `"medium"`, `"high"`    |

---
## OpenRouter
> ref: [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys) | [Model list](https://openrouter.ai/models?max_price=0)
```python
# bash: export OPENROUTER_API_KEY=...
llm = LangChainFactory.create(
    "openrouter::openrouter/free",  # auto-picks free model matching your request
)
llm = LangChainFactory.create(
    "openrouter::meta-llama/llama-3.3-70b-instruct:free",
)
```
* `openrouter/free`: auto-router — picks a free model that supports your request features (tools, vision, etc.)
* **50 req/day** hard cap (shared across all free). $10 lifetime topup → 1000/day

---

## Ollama
-  No key needed for local
```python
# bash: export OLLAMA_API_KEY=...
llm = LangChainFactory.create(
    "ollama::qwen3:4b",
    model_kwargs={"num_ctx": 16384},
)
```

| Param (via `model_kwargs`) | What |
|----------------------------|------|
| `num_ctx` | Context window (default 2048 — set explicitly!) |
| `num_gpu` | GPU layers to offload |

---
## ⭐ Common Config (all providers)
| Param | What | Typical |
|-------|------|---------|
| `temperature` | Randomness | 0 for tools/structured, 0.7 for creative |
| `max_tokens` | Output token cap | 4096–8192. Higher for reasoning (thinking consumes tokens on Groq) |
| `max_retries` | Retry count (default 0) | 0 for free tiers |
| `timeout` | Request timeout (sec) | 30 default; 60–90 for thinking models |
| `seed` | Deterministic output | Set int for reproducibility |
| `model_kwargs` | Provider-specific extras | `{"num_ctx": 16384}` for Ollama |

---
## Tools
> Source: [tools.py](https://github.com/tctsung/free-lunch-ai/blob/main/src/free_lunch/tools.py) with some useful built-in

```python
from free_lunch import build_langchain_tools, web_search, fetch_url
# nice build-in: convert plain functions → LangChain tools
tools = build_langchain_tools(web_search, fetch_url)
llm_with_tools = llm.bind_tools(tools)
```

| Tool | What | Returns |
|------|------|---------|
| `web_search(query, max_results=5)` | DuckDuckGo search | `[{title, url, snippet}]` |
| `fetch_url(url)` | Extract page content | `{url, content}` |
| `current_time(timezone=None)` | Current date/time | `{date, weekday, time, timezone}` |

---

## Using the Model
```python
from pydantic import BaseModel
from langchain_core.tools import tool
from free_lunch import content_blocks_dict

# structured output
class Joke(BaseModel):
    setup: str
    punchline: str
result = llm.with_structured_output(Joke).invoke("Tell me a joke")

# tool calling
@tool
def search(query: str) -> str:
    """Search the web."""
    return "results..."
response = llm.bind_tools([search]).invoke("What's the weather?")
response.tool_calls  # [{"name": "search", "args": {...}}]

# extract reasoning (thinking models)
result = content_blocks_dict(response)
# {"text": "...", "model_id": "...", "reasoning": "..."}
```

---

## ⭐ Suggested Models
> [cheahjs/free-llm-api-resources](https://github.com/cheahjs/free-llm-api-resources) |
> [console.groq.com/docs/rate-limits](https://console.groq.com/docs/rate-limits)

### High quota (agents, multi-step)
| `provider::model`                                     | RPD    | TPM | RPM | Best for                |
| ----------------------------------------------------- | ------ | --- | --- | ----------------------- |
| **`google::gemma-3-27b-instruct`**                    | 14,400 | 15K | 30  | Volume + quality        |
| `google::gemma-3-12b-instruct`                        | 14,400 | 15K | 30  | Volume + lighter        |
| `groq::llama-3.1-8b-instant`                          | 14,400 | 6K  | 30  | Fastest, parsing        |
| **`groq::meta-llama/llama-4-scout-17b-16e-instruct`** | 1,000  | 30K | 30  | Long context            |
| **`groq::qwen/qwen3-32b`**                            | 1,000  | 6K  | 60  | Thinking + multilingual |
| `groq::llama-3.3-70b-versatile`                       | 1,000  | 12K | 30  | General purpose         |
| **`groq::openai/gpt-oss-120b`**                       | 1,000  | 8K  | 30  | Best reasoning          |
| `groq::openai/gpt-oss-20b`                            | 1,000  | 8K  | 30  | Fast reasoning          |
| `google::gemini-3.1-flash-lite`                       | 500    | —   | 15  | Quality fallback        |

### Quality picks (lower quota)
| `provider::model`          | Quota            | Notes                           |
| -------------------------- | ---------------- | ------------------------------- |
| `google::gemini-2.5-pro`   | 5 RPM, ~100/day  | Best free reasoning             |
| `google::gemini-2.5-flash` | 10 RPM, 20/day   | Top flash                       |
| **`groq::groq/compound`**  | 250/day, 70K TPM | Built-in web search + code exec |
| `groq::groq/compound-mini` | 250/day, 70K TPM | Lighter compound                |
| `openrouter::openrouter/free` | 50/day (shared) | Auto-picks best free model per request |

### Avoid for agents
| Provider                        | Why                   |
| ------------------------------- | --------------------- |
| `openrouter::*:free` (no topup) | 50 req/day hard cap   |
| `google::gemini-3.5-flash`      | Only 20 req/day       |
| Cohere                          | 1,000 req/month total |
