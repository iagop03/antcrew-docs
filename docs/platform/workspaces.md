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

Each workspace has one or more API keys. Use them in the `X-Api-Key` header for all REST API requests:

```bash
curl https://antcrew.org/run/ \
  -H "X-Api-Key: acw_live_..." \
  -H "Content-Type: application/json" \
  -d '{"team": "DevTeam", "request": "..."}'
```

Pass the key to `EventBusBridge` when streaming engine events to the platform:

```python
from antcrew_engine.engine import EventBusBridge

bridge = EventBusBridge(
    platform_url="https://antcrew.org",
    api_key="acw_live_...",
    run_id="run_abc123",
)
```

Keys can be rotated at any time from **Settings → API Keys** without downtime.

## Provider keys (BYOK)

Add your own LLM provider API keys in **Settings → Providers**. Keys are encrypted at rest. The proxy uses them to make calls on your behalf — your application code never sees them.
