# Runs & Tickets

## Runs

A **run** is the top-level unit of work. It maps to one execution of an agent pipeline.

### Create a run

```http
POST /api/runs
X-API-Key: your-key

{
  "pipeline_def": "summarise-and-file",
  "input": { "document_id": "doc_123" },
  "template_id": "optional-template-id"
}
```

Response includes `id`, `status`, and a WebSocket URL for live updates.

### Run statuses

| Status | Meaning |
|---|---|
| `pending` | Queued, not yet started |
| `running` | Actively executing |
| `waiting_hitl` | Paused at a HITL checkpoint |
| `completed` | Finished successfully |
| `failed` | Ended with an error |

## Tickets

Tickets are structured action items extracted from run output.

### Display IDs

Each workspace has a configurable prefix (e.g. `PROJ`). Tickets get sequential IDs: `PROJ-00001`, `PROJ-00002`, etc.

Configure the prefix in **Settings → Ticket Settings**.

### Linking to GitHub

If a workspace has a GitHub repo configured, the ticket detail view shows linked commits and PRs that contain the ticket display ID in their message.
