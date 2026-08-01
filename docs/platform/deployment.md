# Deployment

## Environments

| Environment | URL | Trigger |
|---|---|---|
| **INT** | `platform-int.antcrew.org` | Auto on every push to `main` |
| **UAT** | `platform-uat.antcrew.org` | Manual dispatch → Deploy → `uat` |
| **PROD** | _(coming soon)_ | Manual dispatch → Deploy → `prod`, requires UAT gate |

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
