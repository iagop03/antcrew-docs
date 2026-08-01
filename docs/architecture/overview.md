# System overview

## Components

```mermaid
graph TB
    subgraph Client
        SDK[antcrew-engine SDK]
        UI[Browser / dashboard]
    end

    subgraph antcrew-proxy
        PX[OpenAI-compatible proxy]
    end

    subgraph antcrew-platform
        API[FastAPI]
        WS[WebSocket]
        DB[(PostgreSQL)]
    end

    subgraph Providers
        OAI[OpenAI]
        ANT[Anthropic]
        GRQ[Groq]
    end

    SDK -->|POST /runs, events| API
    SDK -->|model calls| PX
    UI -->|REST + WS| API
    UI --> WS
    PX -->|key lookup| API
    PX --> OAI & ANT & GRQ
    API --> DB
    WS --> DB
```

## Environments

| Environment | URL | Infrastructure |
|---|---|---|
| INT | `platform-int.antcrew.org` | Fly.io, auto-deploy on push to main |
| UAT | `platform-uat.antcrew.org` | Hetzner CX22, on-demand, self-deletes |
| PROD | `antcrew.org` | Fly.io, requires UAT gate |
