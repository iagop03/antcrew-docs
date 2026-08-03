# Getting started

## 1. Register and verify your email

Go to `/onboard` and create an account. A 6-digit code is sent to your email — enter it to verify. Once verified you land on the dashboard.

If you're self-hosting, the onboarding wizard also guides you through:
- Choosing an LLM mode (Managed / BYOK / Proxy)
- Adding provider keys if using BYOK

## 2. Your first run

Every workspace comes with an API key shown in **Settings → API Keys**. Use it in the `X-Api-Key` header:

```bash
# Start a pipeline run
curl -X POST https://platform-int.antcrew.org/run/ \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: acw_..." \
  -d '{"team": "DevTeam", "request": "Build a user authentication module"}'

# → {"run_id": "abc123", "status": "running"}
```

Or with the Python SDK:

```python
import httpx

client = httpx.Client(
    base_url="https://platform-int.antcrew.org",
    headers={"X-Api-Key": "acw_..."},
)

r = client.post("/run/", json={
    "team": "DevTeam",
    "request": "Build a user authentication module",
})
run_id = r.json()["run_id"]
```

## 3. Poll or stream

```bash
# Poll status
curl https://platform-int.antcrew.org/runs/abc123 -H "X-Api-Key: acw_..."

# Stream events (Server-Sent Events)
curl -N https://platform-int.antcrew.org/runs/abc123/events -H "X-Api-Key: acw_..."

# WebSocket (all runs)
wscat -c wss://platform-int.antcrew.org/ws/events \
  -H "X-Api-Key: acw_..."
```

## 4. Retrieve artifacts

```bash
curl https://platform-int.antcrew.org/runs/abc123/artifacts -H "X-Api-Key: acw_..."

# Download as ZIP
curl -o artifacts.zip \
  https://platform-int.antcrew.org/runs/abc123/artifacts.zip \
  -H "X-Api-Key: acw_..."
```

## 5. Monitor in the dashboard

Open the platform URL in a browser. The **Runs** tab shows live status, agent event timelines, and artifact browsers. The **Reviews** tab is where HITL decisions happen.

---

## Auth flow reference

| Step | Endpoint | Notes |
|---|---|---|
| Register | `POST /auth/register` | Creates user + workspace, returns admin API key |
| Verify email | `POST /auth/verify-email` | 6-digit code sent on register |
| Resend code | `POST /auth/resend-code` | Re-sends verification code |
| Login | `POST /auth/login` | Returns session cookie; if MFA enabled, returns `mfa_token` instead |
| MFA challenge | `POST /auth/mfa/challenge` | `{mfa_token, code}` → sets session cookie |
| Session info | `GET /auth/me` | Returns `{email, role, email_verified, mfa_enabled}` |

Browser sessions use a `antcrew_session` cookie. API clients use `X-Api-Key`.

---

## LLM modes

| Mode | Who pays | Price multiplier | How to configure |
|---|---|---|---|
| **Managed** | Platform provides the key | ×3.0 | Default — no configuration needed |
| **BYOK** | Your own provider key | ×0.4 | Settings → Providers → add key |
| **Proxy** | Your key, via `antcrew-proxy` | ×0.7 | Run `antcrew-proxy` locally; Settings → Providers → set proxy URL |

BYOK and Proxy keys are encrypted at rest (Fernet AES-128) and never stored in plaintext. See the [configuration reference](configuration.md) for `BYOK_ENCRYPTION_KEY`.
