# Getting started

## 1. Create your account

Go to [antcrew.org/onboard](https://antcrew.org/onboard) and create an account. A 6-digit code is sent to your email — enter it to verify. Once verified you land on the dashboard.

The onboarding wizard guides you through:

- Choosing an LLM mode (Managed / BYOK / Proxy)
- Adding provider keys if using BYOK

## 2. Get your API key

Every workspace comes with an API key shown in **Settings → API Keys**. Use it in the `X-Api-Key` header for all requests.

## 3. Your first run

```bash
# Start a pipeline run
curl -X POST https://antcrew.org/run/ \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: acw_..." \
  -d '{"team": "DevTeam", "request": "Build a user authentication module"}'

# → {"run_id": "abc123", "status": "running"}
```

Or with the Python SDK:

```python
import httpx

client = httpx.Client(
    base_url="https://antcrew.org",
    headers={"X-Api-Key": "acw_..."},
)

r = client.post("/run/", json={
    "team": "DevTeam",
    "request": "Build a user authentication module",
})
run_id = r.json()["run_id"]
```

## 4. Stream events

```bash
# Poll status
curl https://antcrew.org/runs/abc123 -H "X-Api-Key: acw_..."

# Stream events (Server-Sent Events)
curl -N https://antcrew.org/runs/abc123/events -H "X-Api-Key: acw_..."

# WebSocket (all runs)
wscat -c wss://antcrew.org/ws/events -H "X-Api-Key: acw_..."
```

## 5. Monitor in the dashboard

Open [antcrew.org](https://antcrew.org) in a browser. The **Runs** tab shows live status, agent event timelines, and generated tickets. The **Reviews** tab is where HITL decisions happen.

---

## Auth flow reference

| Step | Endpoint | Notes |
|---|---|---|
| Register | `POST /auth/register` | Creates user + workspace |
| Verify email | `POST /auth/verify-email` | 6-digit code sent on register |
| Resend code | `POST /auth/resend-code` | Re-sends verification code |
| Login | `POST /auth/login` | Returns session cookie; if MFA enabled, returns `mfa_token` instead |
| MFA challenge | `POST /auth/mfa/challenge` | `{mfa_token, code}` → sets session cookie |
| Session info | `GET /auth/me` | Returns `{email, role, email_verified, mfa_enabled}` |

Browser sessions use an `antcrew_session` cookie. API clients use `X-Api-Key`.

---

## LLM modes

| Mode | Who provides the key | Price multiplier | How to configure |
|---|---|---|---|
| **Managed** | antcrew provides the key | ×3.0 | Default — no configuration needed |
| **BYOK** | Your own provider key | ×0.4 | Settings → Providers → add key |
| **Proxy** | Your key, via `antcrew-proxy` | ×0.7 | Run `antcrew-proxy`; Settings → Providers → set proxy URL |

BYOK and Proxy keys are encrypted at rest and never stored in plaintext.
