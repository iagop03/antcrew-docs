# Providers

antcrew (bundled engine) supports 16 LLM providers through a single string prefix. The full list is also available at runtime via `GET /engine/supported-providers`.

## Supported providers

| Prefix | Provider | BYOK key | Example model string |
|---|---|---|---|
| `claude:` | Anthropic Claude | `anthropic` | `claude:claude-sonnet-5` |
| `openai:` | OpenAI | `openai` | `gpt-4o` |
| `gemini:` | Google Gemini | `gemini` | `gemini:gemini-2.0-flash` |
| `groq:` | Groq | `groq` | `groq:llama-3.1-70b-versatile` |
| `deepseek:` | DeepSeek | `deepseek` | `deepseek:deepseek-chat` |
| `mistral:` | Mistral AI | `mistral` | `mistral:mistral-large-latest` |
| `xai:` | xAI Grok | `xai` | `xai:grok-3` |
| `together:` | Together AI | `together` | `together:meta-llama/Llama-3-70b-chat-hf` |
| `fireworks:` | Fireworks AI | `fireworks` | `fireworks:accounts/fireworks/models/llama-v3p1-70b-instruct` |
| `cerebras:` | Cerebras | `cerebras` | `cerebras:llama3.1-70b` |
| `moonshot:` | Moonshot AI | `moonshot` | `moonshot:moonshot-v1-8k` |
| `azure:` | Azure OpenAI | `azure` | `azure:gpt-4o` |
| `ollama:` | Ollama (local) | — | `ollama:llama3.2` |
| `lmstudio:` | LM Studio (local) | — | `lmstudio:lmstudio-community/Meta-Llama-3.1-8B-Instruct-GGUF` |
| `vllm:` | vLLM (local) | — | `vllm:meta-llama/Meta-Llama-3-8B-Instruct` |
| `simulated` | Simulated (testing) | — | `simulated` |

Providers with a **BYOK key** require an API key stored in your workspace's LLM keys (`PATCH /workspaces/{id}/llm-keys`). Local providers (Ollama, LM Studio, vLLM) and `simulated` need no key.

## Switching providers

```python
from antcrew_engine.config import build_llm

llm = build_llm("claude:claude-sonnet-5")       # Anthropic
llm = build_llm("openai:gpt-4o")               # OpenAI
llm = build_llm("groq:llama-3.1-70b-versatile") # Groq
llm = build_llm("deepseek:deepseek-reasoner")   # DeepSeek R1
llm = build_llm("ollama:llama3.2")             # local Ollama
llm = build_llm("simulated")                   # deterministic mock
```

## BYOK via the platform

When using antcrew-platform, store your API keys in workspace settings (**Settings → LLM keys**) rather than in environment variables. The platform injects the correct key at run time based on the model prefix.

```bash
# Store a Groq key for the workspace
curl -X PATCH https://antcrew.org/workspaces/42/llm-keys \
  -H "X-Api-Key: acw_..." \
  -H "Content-Type: application/json" \
  -d '{"groq": "gsk_..."}'
```

Once stored, any run in the workspace can use `groq:*` models without setting `GROQ_API_KEY` in the environment.

## BYOK via the proxy

When using keybridge, your keys are stored in the proxy and injected at request time. Application code never handles credentials directly:

```python
llm = build_llm("openai:gpt-4o", base_url="https://proxy.antcrew.org")
# proxy injects the real key — your code never handles credentials
```

## Discover which providers are active for your workspace

The model selector in the **Discover** page automatically filters to providers your workspace has keys for. From the API:

```bash
GET /engine/supported-providers   # full list (16 providers)
GET /workspaces/{id}              # includes byok_providers: ["groq", "deepseek", ...]
```

`byok_providers` lists which BYOK keys are configured. Cross-reference with the `byok_key` field from `/engine/supported-providers` to know which prefixes are ready to use.
