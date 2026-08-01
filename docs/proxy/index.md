# Proxy — overview

`antcrew-proxy` is an OpenAI-compatible HTTP proxy that sits between your application and LLM providers. It handles key injection, request logging, cost tracking, and provider routing.

## Why use the proxy?

- **BYOK without touching keys in code** — keys live in the platform, not in your app
- **Drop-in replacement** — change `base_url` only, no other code changes
- **Provider routing** — route `openai:*` to OpenAI, `anthropic:*` to Anthropic, etc.
- **Observability** — every request is logged with tokens, cost, and latency

## Quick start

```python
import openai

client = openai.OpenAI(
    base_url="https://proxy.antcrew.org/v1",
    api_key="your-antcrew-workspace-key",  # platform key, not OpenAI key
)

# Works exactly like the OpenAI SDK
response = client.chat.completions.create(
    model="openai:gpt-4o-mini",
    messages=[{"role": "user", "content": "Hello"}],
)
```
