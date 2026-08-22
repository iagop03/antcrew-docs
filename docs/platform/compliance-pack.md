# Compliance Pack

antcrew ships the building blocks for compliance-grade AI pipelines. This page shows how to combine them for healthcare, fintech, and legaltech use cases where every agent decision must be auditable, reproducible, and cryptographically linked to its model configuration.

---

## What the compliance infrastructure covers

| Requirement | antcrew feature | How it works |
|---|---|---|
| Full decision audit trail | TraceLog | Every agent call (prompt, response, tokens, cost) written to local SQLite. `antcrew inspect <id>` shows the full trace offline. |
| Reproducibility | Trace replay | `antcrew trace replay <id>` reruns every agent call and compares output — detects model drift between runs. |
| Configuration binding | Governance hash | Each agent configuration produces a deterministic SHA-256 hash. Same config = same hash, always. Cite it in papers, pin it in CI. |
| Run attestation | `POST /run/attest` | Platform generates a signed attestation for any run: model used, governance hash, cost, agent count, outcome. |
| BYOK key isolation | keybridge | LLM API keys stay on your infrastructure. The platform never sees plaintext keys. |
| Encryption at rest | Fernet field encryption | Per-workspace LLM keys and proxy tokens encrypted with AES-128-CBC. Key stored in environment, never in DB. |
| Data residency | Self-hosted deploy | Full Docker + PostgreSQL deploy on your own Hetzner/AWS/Azure instance. Zero external data egress. |
| GDPR Art. 28 | DPA | The platform SLA includes a full Data Processing Agreement with antcrew as Processor and the Customer as Controller. |
| keybridge audit log | Append-only JSONL | Every LLM call through keybridge logged with SHA-256 hashed key, timestamp, provider, status, tokens. Never stores prompt/response content. |

---

## Quick setup — compliance-grade local run

```python
from antcrew import ReproducibleResearchPipeline

# Every run gets: experiment_id = "<team_governance_hash>:<run_id>"
# Both parts are stable — same config = same hash
pipeline = ReproducibleResearchPipeline(db_path="experiments.db")

result = pipeline.run("Analyze contract clauses for GDPR Art. 9 compliance")

print(result.experiment_id)
# → "sha256:a3f4b9...:<run-id>"  — cite this in your audit report

# Replay 30 days later to verify model hasn't drifted
for call in pipeline.replay_experiment(result.experiment_id):
    print(call["agent_name"], "| matched:", call["matched"])
```

---

## Trace inspection

Every antcrew run produces a local SQLite trace, regardless of whether you use the platform:

```bash
# After any run
antcrew inspect <run-id>
```

Output includes per agent:

```
Agent: BackendDev
  model:          claude-sonnet-4-6
  governance_hash: sha256:a3f4b9c1...
  prompt_tokens:  1842
  completion_tokens: 644
  cost_usd:       0.0124
  duration_s:     4.21
  timestamp:      2026-08-22T14:32:11Z
```

```bash
# Verify nothing changed between runs
antcrew trace replay <run-id>
# → Replays all calls and prints per-agent match/drift status
```

---

## Run attestation (platform)

For runs executed via antcrew-platform, request a signed attestation:

```bash
curl -X POST https://app.antcrew.ai/run/attest \
  -H "X-Api-Key: $PLATFORM_API_KEY" \
  -d '{"run_id": "<run-id>"}'
```

Response:

```json
{
  "run_id": "...",
  "attested_at": "2026-08-22T14:32:15Z",
  "model": "claude-sonnet-4-6",
  "governance_hash": "sha256:a3f4b9c1...",
  "agent_count": 7,
  "cost_usd": 0.43,
  "outcome": "completed",
  "signature": "sha256:..."
}
```

Store the attestation alongside your output artifacts. If you need to prove what model produced a given output, replay the run and compare hashes.

---

## BYOK key isolation (keybridge)

For organizations that cannot expose LLM API keys to any third-party platform:

```
antcrew-platform  →  keybridge (your infra)  →  api.anthropic.com
      ↑                     ↑
 sends UUID token      holds your keys
 (never sees keys)     never leaves your network
```

Deploy keybridge on your own infrastructure. The platform sends a rotating UUID token per request — your keys never transit antcrew's servers.

→ [keybridge setup](../proxy/index.md)

---

## Field encryption

Per-workspace LLM API keys (BYOK mode) and keybridge tokens are encrypted at rest using Fernet (AES-128-CBC). The encryption key is an environment variable on your deployment — never stored in the database.

```bash
# Generate a Fernet key for your deployment
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Set it in your deploy environment
BYOK_ENCRYPTION_KEY=<generated-key>
```

→ [Field encryption reference](encryption.md)

---

## Self-hosted deployment

For complete data sovereignty — no external egress, your own Postgres, your own keys:

```bash
docker run -p 8000:8000 \
  -e DATABASE_URL=postgresql+asyncpg://user:pass@your-db/antcrew \
  -e BYOK_ENCRYPTION_KEY=<your-fernet-key> \
  -e ANTHROPIC_API_KEY=<your-key> \
  ghcr.io/iagop03/antcrew-platform:latest
```

All run state, artifacts, and tickets stay in your PostgreSQL instance. antcrew does not retain or replicate data from self-hosted deployments.

---

## Compliance checklist

| Use case | Required components | Notes |
|---|---|---|
| **Healthcare (HIPAA)** | Self-hosted + BYOK/keybridge + TraceLog | No PHI should enter LLM prompts. Use Ollama for fully on-premises inference. |
| **Fintech (SOC 2)** | Attestation + TraceLog + field encryption | Governance hash provides model-config binding for audit. |
| **Legaltech** | ReproducibleResearchPipeline + attestation | Cite experiment_id in legal research. Replay to verify findings. |
| **GDPR Art. 28** | Self-hosted or managed + DPA | Managed instance: DPA executed with antcrew as Processor. Self-hosted: you are Controller and Processor. |
| **General audit trail** | TraceLog (included by default) | Works offline. No platform account needed. |

---

## GDPR / data processing

The antcrew-platform managed instance includes a Data Processing Agreement (DPA) under GDPR Art. 28, with antcrew acting as Processor and the Customer as Controller. Key points:

- Platform hosted in EU (Hetzner Frankfurt/Helsinki)
- Neon PostgreSQL in EU region
- LLM providers (Anthropic, OpenAI, Groq) are sub-processors in the US — Customer must execute provider DPAs
- keybridge mode: LLM calls do not transit antcrew servers; Customer is solely responsible for those transfers
- Right to erasure: `run.request` retained indefinitely by default — Customer (Controller) must implement deletion per their retention policy

For the full DPA, contact legal@antcrew.org.
