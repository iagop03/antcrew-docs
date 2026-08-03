# Security

## Authentication model

antcrew-platform supports two authentication paths:

| Path | Used by | Credential |
|---|---|---|
| **Session cookie** | Browser users (dashboard, onboarding) | `antcrew_session` cookie set on login |
| **API key** | CLI, SDK, CI pipelines | `X-Api-Key` header or `PLATFORM_API_KEY` env var |

The backend (`get_workspace_context`) tries session first, then API key header, then the platform master key.

### Email + password registration

1. `POST /auth/register` — creates user + workspace, sends a 6-digit verification code by email
2. `POST /auth/verify-email` — activates the account
3. `POST /auth/login` — sets `antcrew_session` cookie; if MFA is enabled, returns a short-lived `mfa_token` instead
4. `POST /auth/mfa/challenge` — exchanges `mfa_token + TOTP code` for a full session cookie

## Multi-factor authentication (TOTP)

Users can enable TOTP-based MFA from **Settings → Profile** or the post-onboarding setup prompt. MFA is per-user and optional; workspace admins cannot currently enforce it for all members.

Setup flow:
1. `GET /auth/mfa/setup` — returns a TOTP secret and `otpauth://` provisioning URI (scan with Authenticator app)
2. `POST /auth/mfa/enable` — submits the secret and a verified 6-digit code to activate; requires CSRF token

Login flow when MFA is active:
1. `POST /auth/login` — returns `{mfa_required: true, mfa_token: "..."}` instead of setting a session cookie
2. `POST /auth/mfa/challenge` — submits `{mfa_token, code}`; sets the session cookie on success

MFA can be disabled at any time via `POST /auth/mfa/disable` (requires CSRF token and active session).

## API key model

- Each workspace has one or more **workspace API keys** with a role (`admin | write | read | reviewer`)
- A single key can be a member of multiple workspaces
- Keys can be rotated at any time without downtime
- The platform master key (`PLATFORM_API_KEY`) is a static env var that bypasses the DB check — useful for infrastructure scripts; restrict its exposure

## LLM key storage (BYOK)

Per-workspace LLM provider keys are encrypted at rest using Fernet (AES-128-CBC) under `BYOK_ENCRYPTION_KEY`. The encryption key is an environment variable on the Fly/Cloud Run instance — it is never transmitted over the network.

A single `BYOK_ENCRYPTION_KEY` protects all workspaces on the same instance. For workspaces requiring HSM-grade isolation, self-host with a KMS integration.

## CSRF protection

All state-changing browser endpoints (enable/disable MFA, workspace settings, etc.) require a CSRF token:
- Cookie: `csrf_token`
- Header: `X-CSRF-Token`

API key requests skip CSRF checks.

## HITL as a security gate

Human-in-the-loop reviews prevent unreviewed AI outputs from reaching production systems. Any run can be configured to require approval before its output is acted on. Public review links (`/reviews/token/{token}`) are single-use and scoped to a specific review.

## Audit log

Every review decision (approve/reject), every run state change, and every settings modification is written to an append-only audit log with actor, timestamp, and diff.

## Admin bootstrap

Platform admin access (`is_platform_admin = true`) is granted via:

```bash
curl -X POST https://your-platform.example.com/admin/make-admin \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PLATFORM_ADMIN_TOKEN" \
  -d '{"email": "you@example.com"}'
```

`PLATFORM_ADMIN_TOKEN` is an environment variable you set at deploy time. It is only needed once per environment (or whenever you need to grant admin to a new user). Platform admins can access `/admin` in the dashboard.

## CI/CD secrets reference

| Secret | Scope | Used by |
|---|---|---|
| `ANTHROPIC_API_KEY` | LLM inference | Runtime — required |
| `BYOK_ENCRYPTION_KEY` | Per-workspace key encryption | Runtime — required for BYOK |
| `PLATFORM_ADMIN_TOKEN` | Admin bootstrap | Runtime — set once |
| `PLATFORM_API_KEY` | Master API key (optional) | Runtime |
| `SMTP_HOST` / `SMTP_USER` / `SMTP_PASSWORD` | Email sending | Runtime — required for email |
| `HCLOUD_TOKEN` | Hetzner API | `deploy.yml` — create/delete UAT server |
| `HETZNER_SSH_PRIVATE_KEY` | SSH into UAT server | `deploy.yml` |
| `UAT_DATABASE_URL` | PostgreSQL connection | UAT server runtime |
| `UAT_SECRET_KEY` | App session signing | UAT server runtime |
| `FLY_API_TOKEN` | Fly.io deploy | `deploy.yml` — INT deploy |
| `CLOUDFLARE_TOKEN` | DNS API | `deploy.yml` — update platform-uat DNS |

See the full [configuration reference](../platform/configuration.md) for all variables.
