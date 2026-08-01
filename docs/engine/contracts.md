# Typed contracts

A **contract** is a Python function decorated with `@contract`. The function signature defines the LLM's input schema and expected output type. antcrew-engine enforces both at runtime.

## Basic contract

```python
from antcrew import contract
from pydantic import BaseModel

class SummaryResult(BaseModel):
    title: str
    summary: str
    key_points: list[str]

@contract
def summarise_document(content: str, max_points: int = 5) -> SummaryResult:
    """
    Summarise the document and extract the most important points.
    Return no more than {max_points} key points.
    """
    ...
```

## Chaining contracts

```python
@contract
def translate(text: str, target_language: str) -> str:
    """Translate the text to {target_language}."""
    ...

# Chain: summarise → translate
agent = Agent(model="openai:gpt-4o")
summary = agent.run(summarise_document, content=doc)
translated = agent.run(translate, text=summary.summary, target_language="Spanish")
```

## Validation

If the LLM returns a response that doesn't match the output schema, antcrew-engine retries automatically (up to `max_retries` attempts) before raising `ContractValidationError`.
