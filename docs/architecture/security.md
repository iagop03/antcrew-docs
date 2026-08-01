# Security

## API key model

- Each workspace has one or more **workspace API keys** — these are the credentials your application uses
- LLM provider keys are stored encrypted per-workspace (BYOK) and only decrypted at proxy request time
- Platform uses short-lived **session tokens** for browser authentication

## HITL as a security gate

Human-in-the-loop reviews prevent unreviewed AI outputs from reaching production systems. Any run can be configured to require approval before its output is acted on.

## Audit log

Every review decision (approve/reject), every run state change, and every settings modification is written to an append-only audit log with actor, timestamp, and diff.

## Secrets in CI/CD

| Secret | Scope | Used by |
|---|---|---|
| `HCLOUD_TOKEN` | Hetzner API | deploy.yml — create/delete UAT server |
| `HETZNER_SSH_PRIVATE_KEY` | SSH | deploy.yml — SSH into UAT server |
| `UAT_DATABASE_URL` | PostgreSQL | UAT server runtime |
| `UAT_SECRET_KEY` | App signing | UAT server runtime |
| `FLY_API_TOKEN` | Fly.io | deploy.yml — INT/PROD deploy |
| `CLOUDFLARE_TOKEN` | DNS API | deploy.yml — update platform-uat DNS |
