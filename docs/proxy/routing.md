# Provider routing

The proxy routes requests to the correct provider based on the `model` field prefix.

## How routing works

```mermaid
graph LR
    A[Request: model=openai:gpt-4o] --> B{Parse prefix}
    B -->|openai:| C[Inject OpenAI key]
    B -->|anthropic:| D[Inject Anthropic key]
    B -->|groq:| E[Inject Groq key]
    C & D & E --> F[Forward to provider API]
    F --> G[Return response]
```

The proxy strips the prefix before forwarding so the provider receives the bare model name (`gpt-4o`, `claude-sonnet-5`, etc.).

## Fallback routing

If a prefix is not recognised, the request is forwarded to the OpenAI-compatible endpoint configured as the default provider.

## Per-workspace keys

Each workspace can have different keys for each provider. The proxy resolves the correct key by workspace API key → workspace → provider key lookup.

```
Request: X-API-Key: ws_abc123, model: anthropic:claude-sonnet-5
  → look up workspace for ws_abc123
  → find workspace's Anthropic key
  → inject key + forward to api.anthropic.com
```
