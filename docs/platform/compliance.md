# Compliance Pack

antcrew ships with the infrastructure regulated industries need out of the box. This page maps that infrastructure to common compliance requirements.

## What's included

| Feature | What it does | Where to find it |
|---|---|---|
| **Run attestation** | Cryptographically verifiable provenance document for every run | [Attestation](attestation.md) |
| **Governance hashes** | Detects prompt drift; proves identical agent configuration across runs | [Governance](../engine/governance.md) |
| **TraceLog & Replay** | Immutable, replayable event log for every run | [TraceLog](../engine/tracelog.md) |
| **Field encryption** | Encrypt any output field at rest in the run state | [BYOK](byok.md) |
| **BYOK / data residency** | Your keys, your infra — no model vendor ever sees your data | [BYOK](byok.md) |
| **HITL audit trail** | Every human approval decision is logged with actor, timestamp, and payload | [HITL](hitl.md) |

---

## Attestation for auditors

When a regulator or auditor asks "what AI system produced this output?", you give them an attestation file:

```bash
curl -H "X-Api-Key: acw_..." \
     "https://antcrew.org/runs/{run_id}/attestation" \
     -o attestation-{run_id}.json
```

The file is self-contained JSON. It includes the team, every agent that ran, their governance hashes, token usage, cost, and timestamps. The `document_hash` field lets anyone verify the file was not modified after generation — no access to the platform required.

See [Attestation](attestation.md) for the full format and verification scripts.

---

## Governance hashes

A governance hash is a 16-character SHA-256 prefix over an agent's name, role, stage, system prompt suffix, and tool list. It changes if any of those change.

This gives you:

- **Proof of configuration** — "Run 7842 used the approved agent configuration"
- **Drift detection** — a hash mismatch in CI means someone changed a prompt without review
- **Cross-run equivalence** — two runs with the same governance hash are provably identical in agent setup

The hash is in every attestation, in every `agent.end` TraceLog event, and accessible at runtime as `agent.governance_hash`.

---

## TraceLog as an audit trail

Every run produces an immutable event log: `run.start`, `agent.start`, `tool.call`, `tool.result`, `agent.end`, `run.end`. Events are stored in the `events` table and streamed to the client.

Relevant for:

- **GDPR Article 12** — demonstrates automated decision-making with explainable step-by-step logic
- **SOC 2 Type II** — demonstrates that access to AI-generated outputs is logged and attributable
- **Internal audit** — the TraceLog is the full reasoning trail; Studio can replay it interactively

Replay a run via Studio or download the raw events:

```bash
curl -H "X-Api-Key: acw_..." "/runs/{run_id}/events" | jq '.'
```

---

## Field encryption

Sensitive fields in the run state (PII, PHI, confidential outputs) can be encrypted at rest using your own keys. The platform stores ciphertext; plaintext never leaves your environment.

Configure via workspace settings → BYOK → field encryption. See [BYOK](byok.md).

---

## Data residency & BYOK

- **BYOK models** — point antcrew at your own LLM endpoint (Azure OpenAI in your region, Bedrock, on-prem Ollama). The model vendor never sees your data.
- **Self-hosted** — deploy the platform on your infrastructure. antcrew never sees your run payloads.
- **Managed with BYOK keys** — use antcrew's managed service but supply your own encryption keys. We cannot read your run state.

See [BYOK](byok.md) and [Proxy](../proxy/index.md) for setup.

---

## HITL audit trail

Every human-in-the-loop approval decision is recorded:

```json
{
  "event_type": "hitl.approval",
  "actor": "alice@example.com",
  "decision": "approved",
  "run_id": "abc123",
  "timestamp": "2026-08-14T10:05:00+00:00"
}
```

This lets you answer: who approved which AI action, when, and what the action was.

---

## Regulatory mapping

| Regulation | antcrew feature |
|---|---|
| GDPR Art. 12 (transparency in automated decisions) | TraceLog + Attestation |
| GDPR Art. 25 (data protection by design) | BYOK + field encryption |
| HIPAA § 164.312(b) (audit controls) | TraceLog + HITL audit |
| SOC 2 CC7.2 (system monitoring) | TraceLog + governance hashes |
| ISO 27001 A.12.4 (logging & monitoring) | TraceLog + attestation |
| EU AI Act Art. 13 (transparency for high-risk AI) | Attestation + governance hashes |

---

## Checklist for a compliance audit

- [ ] Download attestation for each run in scope: `GET /runs/{run_id}/attestation`
- [ ] Verify `document_hash` integrity with the [verification script](attestation.md#verifying-integrity)
- [ ] If `ATTESTATION_HMAC_SECRET` is configured, verify `hmac_sha256` as well
- [ ] Export TraceLog events: `GET /runs/{run_id}/events`
- [ ] Confirm governance hashes match your approved configuration snapshot
- [ ] Review HITL approval log for any human decisions in the run

---

## Questions

Open an issue or reach the team at [support@antcrew.org](mailto:support@antcrew.org).
