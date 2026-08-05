# Data retention

antcrew-platform stores several types of data with different lifetimes. This page documents
what is kept, for how long, and what that means for compliance.

---

## Retention policies

| Table | Default lifetime | Configurable via | Notes |
|---|---|---|---|
| `event` | 30 days | `DATA_RETENTION_DAYS` | Engine events (agent.start, agent.end, …) |
| `webhook_delivery` | 30 days | `DATA_RETENTION_DAYS` | Only terminal rows (delivered, failed) |
| `discovery_session` | 7 days (last activity) | `DISCOVERY_SESSION_TTL_DAYS` | Deleted by background job |
| `run` | **indefinite** | — | See below |
| `ticket` | **indefinite** | — | Linked to runs |
| `hitl_review` | **indefinite** | — | Linked to runs |

Cleanup runs on background tasks: events and webhook deliveries are swept daily; discovery
sessions are swept every 6 hours.

---

## What `run.request` contains

The `run.request` column stores the exact free-form text submitted by the caller — the
natural-language description of the work to perform. This may include:

- Feature descriptions, bug reports, business requirements
- Code snippets or configuration fragments
- References to internal systems, employees, or customers

**This column is never automatically deleted.** Runs form the audit trail for completed
work and are retained indefinitely by default.

---

## GDPR and LOPDGDD considerations (Spain)

antcrew is operated by a company established in Spain and is subject to the GDPR
(Regulation (EU) 2016/679) and the LOPDGDD (Organic Law 3/2018).

**Key points:**

- `run.request` is potentially personal data if it identifies individuals or can be linked
  to an identified natural person. Under GDPR Art. 5(1)(e), personal data must not be kept
  longer than necessary for the purpose for which it was collected.

- If a user exercises their right to erasure (GDPR Art. 17), the platform does not currently
  have an automated erasure workflow. A platform administrator must delete the relevant rows
  directly in the database (see [Manual deletion](#manual-deletion) below).

- The current indefinite retention of runs is justifiable for operational and audit purposes.
  If your deployment has stricter retention obligations, consider adding a `RUN_RETENTION_DAYS`
  env var to the platform configuration and opening an issue to track the implementation.

- `event` and `webhook_delivery` rows are deleted after `DATA_RETENTION_DAYS` (default: 30).
  These tables may contain run IDs and partial request context in event payloads — the
  30-day window is a reasonable balance between operational observability and data minimisation.

---

## Configuration

Set in `.env.prod` on your Hetzner (or self-hosted) server:

```env
# How long to retain event and webhook_delivery rows (days)
DATA_RETENTION_DAYS=30

# How long to retain idle discovery sessions (days of inactivity)
DISCOVERY_SESSION_TTL_DAYS=7
```

---

## Manual deletion

To delete a specific run and all associated data, execute in order (no cascading FK):

```sql
DELETE FROM event        WHERE run_id = 'run_01...';
DELETE FROM ticket       WHERE run_id = 'run_01...';
DELETE FROM hitl_review  WHERE run_id = 'run_01...';
DELETE FROM run          WHERE run_id = 'run_01...';
```

To wipe all data for a workspace (e.g., on account deletion):

```sql
-- Get all run_ids first
SELECT run_id FROM run WHERE workspace_id = <id>;

-- Then delete in order
DELETE FROM event       WHERE run_id IN (SELECT run_id FROM run WHERE workspace_id = <id>);
DELETE FROM ticket      WHERE run_id IN (SELECT run_id FROM run WHERE workspace_id = <id>);
DELETE FROM hitl_review WHERE run_id IN (SELECT run_id FROM run WHERE workspace_id = <id>);
DELETE FROM run         WHERE workspace_id = <id>;
DELETE FROM workspace   WHERE id = <id>;
```

!!! warning "No cascade"
    SQLModel relationships are not configured with `ondelete="CASCADE"` on the run-level
    tables. Always delete child rows before parent rows to avoid foreign-key violations.
