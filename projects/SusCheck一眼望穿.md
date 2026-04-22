---
created: 2026-03-15T17:56
updated: 2026-04-21T21:18
---
### Intro
a lightweight, asynchronous CLI monitor for tracking agentic workflows by displaying a static, Duolingo-style ASCII face widget in the terminal. The architecture uses an `asyncio.Queue` to ingest execution logs without blocking the main process, bypassing the need to read physical log files. To determine the agent's status, the system first uses a fast-path regex to catch obvious states like errors or loops, and then falls back on a free-tier tiny LLM API to classify ambiguous logs into a predefined dictionary of ASCII facial expressions (such as `( o_o)` for thinking or `( @_@)` for loops). Finally, it uses a carriage return (`\r`) to continuously overwrite a single line at the bottom of the terminal, giving you an instant, glanceable emotional heartbeat of your background tasks without the clutter of scrolling logs.


### note
- CLI monitor mode (input: log file location)
- async mode (embed into langchain, put in logs or some files)



```python
FACES = {
    "idle": "( -_-)",       # Waiting for tasks
    "thinking": "( o_o)",   # Processing / querying
    "focused": "( >_<)",    # Executing tools / heavy computation
    "success": "( ^ᴗ^)",    # Completed task
    "error": "( x_x)",      # API failure / crash
    "loop": "( @_@)",       # Stuck / retrying
    "surprised": "( O_O)",  # Unexpected output / anomaly
    "sad": "( T_T)",        # Rate limited / hard stop
    "angry": "( ò_ó)",       # Critical failure
    "stupid": "( ≖_≖)",
    "sus": "( 〇_o)",
    "wink": "( ^_-)"
}
```




```python
import time, sys

FACES = {
    "waiting": {
        "emoji":     "( -_-)",
        "color":     "#639922",
        "animation": ["( -_-)", "( ._-)", "( -_.)", "( -_-)", "(  z  )", "( -_-)"],
    },
    "thinking": {
        "emoji":     "( o_o)",
        "color":     "#3B6D11",
        "animation": ["( o_o)", "( o_O)", "( O_O)", "( O_o)", "( o_o)"],
    },
    "vibing": {
        "emoji":     "( ^w^)",
        "color":     "#1D9E75",
        "animation": ["( ^w^)", "( ^W^)", "( uwu)", "( >w<)", "( ^w^)"],
    },
    "succeed": {
        "emoji":     "( ^v^)",
        "color":     "#5DCAA5",
        "animation": ["( ^v^)", "( ^V^)", "(  !! )", "( ^-^)", "( ^v^)"],
    },
    "sus": {
        "emoji":     "( o_O)",
        "color":     "#888780",
        "animation": ["( o_O)", "( O_o)", "( -_O)", "( O_-)", "( o_O)"],
    },
    "stuck": {
        "emoji":     "( @_@)",
        "color":     "#EF9F27",
        "animation": ["( @_@)", "( @_-)", "( -_@)", "( @_@)", "( @_-)"],
    },
    "ratelimit": {
        "emoji":     "( $.$)",
        "color":     "#BA7517",
        "animation": ["( $.$)", "( $_ )", "(  _$)", "( $ $)", "( $.$)"],
    },
    "crashed": {
        "emoji":     "( x_x)",
        "color":     "#D85A30",
        "animation": ["   o   ", "( x_x)", "  o    ", "( x_x)", "   o   ", "( x_x)"],
    },
    "critical": {
        "emoji":     "( !_!)",
        "color":     "#791F1F",
        "animation": ["( !_!)", "( -_!)", "( !_-)", "( !!!)", "( !_!)"],
    },
}

_SPEEDS = {
    "waiting":   0.6,
    "thinking":  0.28,
    "vibing":    0.35,
    "succeed":   0.45,
    "sus":       0.38,
    "stuck":     0.20,
    "ratelimit": 0.30,
    "crashed":   0.25,
    "critical":  0.10,
}

def animate(emotion: str, duration: float = 3.0):
    data   = FACES[emotion]
    frames = data["animation"]
    speed  = _SPEEDS.get(emotion, 0.35)
    end    = time.time() + duration
    i      = 0
    try:
        while time.time() < end:
            sys.stdout.write(f"\r  {frames[i % len(frames)]}  {emotion:<10}")
            sys.stdout.flush()
            time.sleep(speed)
            i += 1
    finally:
        sys.stdout.write(f"\r  {data['emoji']}  {emotion:<10}\n")
        sys.stdout.flush()


if __name__ == "__main__":
    for name in FACES:
        animate(name, duration=2.5)
```