# Workspaces & API Keys

## Workspaces

A workspace is an isolated tenant. Everything — runs, tickets, reviews, provider keys — belongs to a workspace.

## Members & roles

| Role | Capabilities |
|---|---|
| `owner` | Full access including billing and member management |
| `admin` | All workspace settings, no billing |
| `member` | Create runs, view tickets, submit reviews |
| `reviewer` | View and action HITL reviews only |

## API Keys

Each workspace has one or more API keys. Use them in the `X-API-Key` header for all API requests and in the antcrew-engine SDK.

```python
agent = Agent(
    model="openai:gpt-4o",
    platform_api_key="acw_live_...",
)
```

Keys can be rotated at any time from **Settings → API Keys** without downtime.

## Provider keys (BYOK)

Add your own LLM provider API keys in **Settings → Providers**. Keys are encrypted at rest. The proxy uses them to make calls on your behalf — your application code never sees them.
