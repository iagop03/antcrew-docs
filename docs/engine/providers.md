# Providers

antcrew-engine supports every major LLM provider through a single string prefix.

## Supported prefixes

| Prefix | Provider | Example model |
|---|---|---|
| `openai:` | OpenAI | `openai:gpt-4o` |
| `anthropic:` | Anthropic | `anthropic:claude-sonnet-5` |
| `groq:` | Groq | `groq:llama-3.3-70b-versatile` |
| `gemini:` | Google Gemini | `gemini:gemini-2.0-flash` |
| `moonshot:` | Moonshot AI | `moonshot:moonshot-v1-8k` |
| `simulated:` | Built-in mock | `simulated:echo` |

## Switching providers

```python
# GPT-4o
agent = Agent(model="openai:gpt-4o")

# Claude — same code, different model string
agent = Agent(model="anthropic:claude-sonnet-5")

# Free local mock for tests
agent = Agent(model="simulated:echo")
```

## BYOK via the Proxy

When using antcrew-proxy, your keys are stored in the platform and injected at request time. Your application code never handles credentials directly:

```python
agent = Agent(
    model="openai:gpt-4o",
    base_url="https://proxy.antcrew.org",  # proxy handles the key
)
```
