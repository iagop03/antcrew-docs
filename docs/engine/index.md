# Engine — overview

`antcrew-engine` is the Python SDK for defining and running agent pipelines. It provides typed contracts, a TraceLog for full replay, and integrations with every major LLM provider.

```bash
pip install antcrew-engine
```

## Hello world

```python
from antcrew import Agent, contract

@contract
def summarise(content: str) -> str:
    """Summarise the given text in one paragraph."""
    ...

agent = Agent(model="openai:gpt-4o-mini")
result = agent.run(summarise, content="Long document here…")
print(result)
```

## Key features

- **Typed contracts** — Python functions as LLM prompts with type-checked inputs/outputs
- **TraceLog** — every token, tool call, and decision is logged for replay
- **Provider-agnostic** — swap models with one string change
- **HITL hooks** — pause execution for human review at any step
