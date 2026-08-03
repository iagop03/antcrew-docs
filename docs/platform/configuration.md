# Configuration reference

All configuration is via environment variables. In development, create a `.env` file at the project root and set them there. In production (Fly.io, Docker, Hetzner), set them as platform secrets.

Only `ANTHROPIC_API_KEY` is required for a local dev run with SQLite.

---

## Core

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `sqlite+aiosqlite:///platform.db` | DB connection string. Use `postgresql+asyncpg://user:pass@host/db` in production. |
| `ANTHROPIC_API_KEY` | — | **Required.** Default LLM key for managed mode. |
| `BASE_URL` | `https://app.antcrew.ai` | Public base URL embedded in invite and join-request emails. |
| `PLATFORM_BASE_URL` | — | Public base URL embedded in HITL review links sent via webhooks/Slack. |
| `ANTCREW_WORKERS` | `4` | Background thread pool size for engine runs. |

## Auth

| Variable | Default | Description |
|---|---|---|
| `PLATFORM_API_KEY` | — | Static master key. When set, any request with this key bypasses the database key check. Useful for initial bootstrap or infrastructure scripts. |
| `BYOK_ENCRYPTION_KEY` | — | Fernet key for encrypting per-workspace LLM API keys and proxy tokens at rest. Generate with `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`. |
| `PLATFORM_ADMIN_TOKEN` | — | Bootstrap token for granting platform admin access. Send `POST /admin/make-admin` with `{"email": "you@example.com"}` and `Authorization: Bearer <token>`. Required only once per environment to create the first admin. |

## Email

Required to send verification codes, workspace invites, and HITL notifications.

| Variable | Default | Description |
|---|---|---|
| `SMTP_HOST` | — | SMTP server hostname. Leave unset to disable all email sending. |
| `SMTP_PORT` | `587` | SMTP port. Use `465` with `SMTP_TLS=ssl`. |
| `SMTP_USER` | — | SMTP login username (usually the sender address). |
| `SMTP_PASSWORD` | — | SMTP login password or app password. |
| `SMTP_FROM` | `SMTP_USER` | From address on outgoing emails. Defaults to `SMTP_USER` if unset. |
| `SMTP_TLS` | `true` | TLS mode: `true` = STARTTLS (port 587), `ssl` = SMTPS (port 465), `none` = plain (not recommended). |

## LLM providers

| Variable | Default | Description |
|---|---|---|
| `OPENAI_API_KEY` | — | OpenAI models and eval judge scoring. |
| `GROQ_API_KEY` | — | Groq inference. |
| `GEMINI_API_KEY` | — | Google Gemini. |

Other providers (Ollama, Moonshot, DeepSeek, etc.) are configured by their model string prefix — see the [engine providers reference](../engine/providers.md).

## HITL integrations

### Slack

| Variable | Default | Description |
|---|---|---|
| `SLACK_BOT_TOKEN` | — | `xoxb-…` bot token. Enables Slack interactive HITL (Socket Mode). |
| `SLACK_APP_TOKEN` | — | `xapp-…` app-level token for Socket Mode. |
| `SLACK_CHANNEL_ID` | — | Default Slack channel for HITL review messages. |
| `SLACK_TOKEN_ENCRYPTION_KEY` | — | Fernet key for encrypting per-workspace Slack tokens at rest. |

### Telegram

| Variable | Default | Description |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | — | Bot token from BotFather. |
| `TELEGRAM_CHAT_ID` | — | Default chat/group ID for notifications. |

## Integrations

| Variable | Default | Description |
|---|---|---|
| `GITHUB_TOKEN` | — | Personal access token for GitHub PR automation (`pipeline.end` → branch → PR). |
| `WEBHOOK_URL` | — | Global fallback webhook fired on every `pipeline.end` event across all workspaces. |

## HITL tuning

| Variable | Default | Description |
|---|---|---|
| `HITL_TIMEOUT_S` | `3600` | Seconds before a pending HITL review auto-times-out and the run resumes/fails. |

---

## Example `.env` for local development

```dotenv
# Minimum required
ANTHROPIC_API_KEY=sk-ant-...

# Optional: use PostgreSQL instead of the default SQLite
# DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/antcrew

# Auth bootstrap (run once to create the first platform admin)
# PLATFORM_ADMIN_TOKEN=your-secret-token

# Email (verification codes, invites — leave unset to skip)
# SMTP_HOST=smtp.example.com
# SMTP_PORT=587
# SMTP_USER=hello@example.com
# SMTP_PASSWORD=
# SMTP_FROM=hello@example.com

# BYOK encryption (required to store per-workspace LLM keys)
# BYOK_ENCRYPTION_KEY=...

# Public URLs (required for invite/review links to work)
# BASE_URL=http://localhost:8000
# PLATFORM_BASE_URL=http://localhost:8000
```

---

## Generating Fernet keys

Both `BYOK_ENCRYPTION_KEY` and `SLACK_TOKEN_ENCRYPTION_KEY` must be valid Fernet keys:

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

Generate a separate key for each variable. Rotating a key requires re-encrypting all stored secrets — there is no automatic migration.
