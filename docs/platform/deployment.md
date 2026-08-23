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

---

## Multi-worker deployments

### Sticky sessions (required)

The SSE broadcaster and WebSocket state are **per-process**. With multiple uvicorn workers (or multiple replicas), a client that connects to worker W1 will only receive events from runs that execute on W1. If the load balancer distributes requests round-robin, clients land on different workers than their run and see silence.

**Sticky sessions must be enabled** on any load balancer or reverse proxy in front of a multi-worker deployment:

=== "nginx"
    ```nginx
    upstream antcrew {
        ip_hash;  # sticky by client IP
        server 127.0.0.1:8000;
        server 127.0.0.1:8001;
    }
    ```

=== "Fly.io (single machine)"
    Fly.io routes each request to the nearest healthy machine; for SSE/WebSocket use a single machine or add `fly-prefer-region` header pinning.

=== "Caddy"
    ```caddy
    reverse_proxy * {
        lb_policy cookie antcrew_sticky
    }
    ```

!!! warning
    Without sticky sessions: runs and SSE clients may land on different workers, causing clients to never receive events. Symptoms: run completes server-side, client shows "running" indefinitely.

### Database connection pool

Each worker opens its own connection pool. The default (5 pool + 5 overflow per worker) means 4 workers → up to 40 connections. Size your PostgreSQL `max_connections` accordingly, or lower the pool via env vars:

| Variable | Default | Notes |
|---|---|---|
| `DB_POOL_SIZE` | `5` | Connections kept open per worker |
| `DB_MAX_OVERFLOW` | `5` | Burst connections allowed per worker |
| `DB_POOL_TIMEOUT` | `30` | Seconds to wait for a connection before error |

Rule of thumb: `DB_POOL_SIZE × workers × replicas < max_connections − 10` (reserve 10 for admin queries).
