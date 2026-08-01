# antcrew

**antcrew** is an AI agent orchestration platform built for teams that need reliable, auditable, human-in-the-loop AI workflows.

## Products

<div class="grid cards" markdown>

- **Platform** — web UI and API for managing workspaces, runs, tickets, and reviews
- **Engine** — typed Python SDK for defining and executing agent contracts
- **Proxy** — OpenAI-compatible proxy with provider routing and BYOK support

</div>

## Architecture at a glance

```mermaid
graph LR
    A[Your code] -->|antcrew-engine SDK| B[antcrew-proxy]
    B -->|routes to| C[OpenAI / Anthropic / Groq / …]
    A -->|POST /runs| D[antcrew-platform API]
    D -->|TraceLog + Replay| E[(PostgreSQL)]
    D -->|HITL review| F[Human reviewer]
```

## Quick links

- [Getting started with Platform](platform/getting-started.md)
- [Typed contracts with Engine](engine/contracts.md)
- [TraceLog & Replay](engine/tracelog.md)
- [Proxy routing](proxy/routing.md)
