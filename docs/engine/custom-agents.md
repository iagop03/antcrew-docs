# Custom agents

Beyond simple contract calls, antcrew-engine lets you define multi-step agents with tools, memory, and conditional branching.

## Agent with tools

```python
from antcrew import Agent, tool, contract

@tool
def search_web(query: str) -> str:
    """Search the web and return the top result."""
    # your implementation
    ...

@tool
def read_file(path: str) -> str:
    """Read a file from disk."""
    with open(path) as f:
        return f.read()

@contract
def research_topic(topic: str) -> str:
    """Research {topic} using available tools and write a concise report."""
    ...

agent = Agent(
    model="openai:gpt-4o",
    tools=[search_web, read_file],
)
report = agent.run(research_topic, topic="quantum computing trends 2025")
```

## HITL pause points

```python
from antcrew import Agent, hitl_checkpoint

agent = Agent(model="anthropic:claude-sonnet-5")

draft = agent.run(write_draft, topic="...")

# Pause here and wait for a human to approve before continuing
approved_draft = hitl_checkpoint(
    value=draft,
    prompt="Please review this draft before it is sent.",
    platform_api_key="...",
)

agent.run(send_email, content=approved_draft)
```

The HITL checkpoint creates a review in the platform dashboard and blocks execution until approved or rejected.
