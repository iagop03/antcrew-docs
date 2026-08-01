# Getting started

## 1. Create a workspace

After logging in, create a workspace. Each workspace has its own API key, members, and LLM provider settings.

## 2. Add an LLM provider key

Go to **Settings → Providers** and add your API keys for OpenAI, Anthropic, Groq, or any other supported provider. These are stored encrypted (BYOK — bring your own key).

## 3. Submit your first run

```python
import httpx

client = httpx.Client(base_url="https://platform-int.antcrew.org")

response = client.post("/api/runs", json={
    "pipeline_def": "my-pipeline",
    "input": {"prompt": "summarise this document", "content": "..."},
}, headers={"X-API-Key": "your-workspace-api-key"})

run_id = response.json()["id"]
print(f"Run started: {run_id}")
```

## 4. Monitor in the dashboard

Open `/runs` to see the run status, events, and generated tickets in real time.
