# Proxy — overview

`antcrew-proxy` (v2.1.0) is an OpenAI-compatible HTTP proxy that sits between antcrew-platform and LLM providers. It handles key injection, multi-provider failover, request auditing, and metrics.

## Why use the proxy?

- **BYOK without key exposure** — provider keys live in your proxy, not in the platform or application code
- **Drop-in replacement** — change `base_url` only; no other code changes required
- **15 providers** — Anthropic, OpenAI, Azure OpenAI, Groq, Gemini, Moonshot, DeepSeek, Mistral, xAI, Together, Fireworks, Cerebras, Ollama, LM Studio, vLLM
- **Automatic failover** — `POST /v1/chat/completions` tries providers in order; recovers from timeouts and rate-limits
- **Observability** — per-request audit log + live metrics at `GET /metrics`
- **Multi-key round-robin** — distribute load across multiple API keys per provider

## Quick start

```python
import openai

client = openai.OpenAI(
    base_url="https://proxy.antcrew.org/openai",
    api_key="your-proxy-token",          # proxy token, not an OpenAI key
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
```

Or use the unified endpoint to let the proxy pick the best available provider:

```python
client = openai.OpenAI(
    base_url="https://proxy.antcrew.org",   # /v1 path
    api_key="your-proxy-token",
)

response = client.chat.completions.create(
    model="any",                             # model is replaced by FAILOVER_CHAIN
    messages=[{"role": "user", "content": "Hello"}],
)
# X-Proxy-Provider response header tells you which provider was used
```

## Endpoints summary

| Endpoint | Description |
|---|---|
| `POST /{provider}/{path}` | Route to a specific provider |
| `POST /v1/chat/completions` | Unified with automatic failover via `FAILOVER_CHAIN` |
| `GET /health` | Liveness check + provider key summary |
| `GET /metrics` | Live in-memory request stats (p50/p99 latency, tokens, errors) |

## Key concepts

- **Auth**: set `x-api-key` or `Authorization: Bearer` to your proxy token (not a provider key)
- **Routing**: path prefix determines the provider — `/anthropic/v1/messages`, `/openai/v1/chat/completions`, etc.
- **Key injection**: the proxy strips your token and injects the real provider key from its environment
- **Audit log**: every request logged to `AUDIT_LOG_PATH` in JSON-lines format; API keys are SHA-256 hashed

See [Provider routing](routing.md) and [Configuration](configuration.md) for the full reference.
