# Deployment

## Environments

| Environment | URL | Trigger |
|---|---|---|
| **INT** | `platform-int.antcrew.org` | Auto on every push to `main` (requires reviewer approval) |
| **UAT** | `platform-uat.antcrew.org` | Manual dispatch → Deploy → `uat` |
| **PROD** | _(coming soon)_ | Manual dispatch → Deploy → `prod`, requires successful UAT gate |

## Deploying to UAT

1. Go to GitHub → Actions → **Deploy**
2. Click **Run workflow**
3. Set **environment** to `uat`, choose the auto-shutdown window (1–8h)
4. The server is created from a snapshot, the app is deployed, and the server self-deletes after the window

## Restarting UAT without redeploying

If the UAT server shut down and you want to test against the same build:

1. GitHub → Actions → **UAT — on-demand start**
2. Run workflow → choose hours

This restarts the existing server and resets the auto-delete timer.

## INT → PROD promotion

Prod deploys require the last UAT deployment to be successful (enforced by the `gate-prod` job in the Deploy workflow). This prevents unvalidated code from reaching production.

---

## First-time setup per environment

### 1. Apply database migrations

Migrations run automatically on deploy via the Alembic `release_command` in `fly.toml`. If you need to run them manually:

```bash
# Fly.io
fly ssh console -a antcrew-int -- bash -c "cd /app && python -m alembic upgrade head"

# Hetzner UAT (SSH)
ssh root@<server-ip> "cd /app && python -m alembic upgrade head"
```

### 2. Bootstrap the first admin

After deploying, promote your account to platform admin so you can access `/admin`:

```bash
curl -X POST https://your-env.antcrew.org/admin/make-admin \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PLATFORM_ADMIN_TOKEN" \
  -d '{"email": "you@example.com"}'
```

`PLATFORM_ADMIN_TOKEN` must be set as a secret on the environment (see below). You only need to run this once per environment.

### 3. Environment secrets

Set these in GitHub → Settings → Environments → `<env name>` → Secrets:

**Runtime (all environments):**

| Secret | Required | Notes |
|---|---|---|
| `ANTHROPIC_API_KEY` | Yes | LLM inference key |
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `BYOK_ENCRYPTION_KEY` | Yes | Fernet key — generate with `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| `PLATFORM_ADMIN_TOKEN` | Yes | Bootstrap token for granting admin access |
| `BASE_URL` | Yes | Public base URL (e.g. `https://platform-int.antcrew.org`) |
| `SMTP_HOST` | If email needed | SMTP server hostname |
| `SMTP_USER` | If email needed | SMTP login username |
| `SMTP_PASSWORD` | If email needed | SMTP password or app password |
| `SMTP_FROM` | No | Defaults to `SMTP_USER` |

**Infrastructure (UAT only):**

| Secret | Notes |
|---|---|
| `HCLOUD_TOKEN` | Hetzner API — create/delete servers |
| `HETZNER_SSH_PRIVATE_KEY` | SSH private key for the UAT server |
| `CLOUDFLARE_TOKEN` | DNS API — update `platform-uat` CNAME |

**Fly.io (INT):**

| Secret | Notes |
|---|---|
| `FLY_API_TOKEN` | Fly.io deploy token |

See the full [configuration reference](configuration.md) for all available variables.

---

## Fly.io manual deploy

```bash
fly apps create antcrew-platform
fly secrets set \
  ANTHROPIC_API_KEY=sk-ant-... \
  DATABASE_URL=postgresql+asyncpg://... \
  BYOK_ENCRYPTION_KEY=... \
  PLATFORM_ADMIN_TOKEN=...
fly deploy
```

The provided `fly.toml` builds a Docker image from `Dockerfile`, runs Alembic migrations on release (`release_command`), and exposes port 8000.
