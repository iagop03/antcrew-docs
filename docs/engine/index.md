# Engine SDK

`antcrew-engine` is the Python library that runs inside your agent code. It is the only component your application imports directly.

```bash
pip install antcrew-engine
```

---

## What it provides

**Typed contracts** — Python functions that compile to LLM prompts. The function signature defines the input schema and expected output type. If the model returns something that doesn't match, the engine retries automatically.

**Provider-agnostic model calls** — change `"openai:gpt-4o"` to `"anthropic:claude-opus-5"` and nothing else changes. The same `@contract` function runs on any provider.

**TraceLog** — every LLM call, tool invocation, retry, and result is written to a structured, append-only log. You can replay any past run exactly, without making new API calls.

**HITL checkpoints** — `hitl_checkpoint()` pauses execution and waits for a human to approve before continuing. The review request is sent to antcrew-platform and the human responds from the dashboard.

---

## Hello world

```python
from antcrew import Agent, contract

@contract
def summarise(content: str) -> str:
    """Summarise the text in one concise paragraph."""
    ...

agent = Agent(model="openai:gpt-4o-mini")
result = agent.run(summarise, content="Long document…")
print(result)
```

No prompt engineering. No JSON parsing. No retry logic. The contract signature is the schema; the docstring is the prompt.

---

## Connecting to the platform

To send TraceLog events to antcrew-platform in real time:

```python
from antcrew import Agent
from antcrew.platform import PlatformSink

agent = Agent(
    model="anthropic:claude-opus-5",
    trace_sink=PlatformSink(
        base_url="https://platform.yourcompany.com",
        api_key="acw_live_...",
    ),
)
```

Every run will appear in the platform dashboard with its full event log, status, and any tickets generated.

---

## Core concepts

| Concept | What it is |
|---|---|
| `@contract` | A Python function that defines an LLM task — signature = schema, docstring = prompt |
| `Agent` | Executes contracts against a configured model and writes to a TraceLog |
| `TraceLog` | Append-only record of every token, tool call, and decision in a run |
| `hitl_checkpoint` | Blocking call that pauses execution for a human to approve |
| `PlatformSink` | Streams TraceLog events to antcrew-platform in real time |

[:octicons-arrow-right-24: Typed contracts — full reference](contracts.md)

[:octicons-arrow-right-24: TraceLog & Replay](tracelog.md)

[:octicons-arrow-right-24: LLM providers](providers.md)
