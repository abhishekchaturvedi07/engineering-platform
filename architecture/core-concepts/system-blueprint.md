# 🏛️ Enterprise System Blueprint & Data Flow

This document outlines the high-level architectural patterns, data flow, and boundaries for the platform. The system is designed for high availability, strict domain isolation, and agentic AI processing.

## 1. Core Architectural Decisions

### A. API Entry: The Gateway + BFF Pattern

- **API Gateway (Edge):** Acts as the perimeter defense. Handles SSL termination, rate limiting, and initial JWT validation.
- **GraphQL BFF (Backend-For-Frontend):** Sits behind the gateway. It stitches together multiple REST responses from downstream microservices into a single, cohesive GraphQL graph for the Next.js frontend to consume efficiently.

### B. Microservice Communication: Sync + Async Hybrid

- **External to Internal (Synchronous):** The frontend communicates with the Gateway/BFF via REST/GraphQL over HTTPS.
- **Internal Service-to-Service (Asynchronous):** Decoupled internal communication is handled via an Event Broker (**Kafka / RabbitMQ**). Services emit domain events (e.g., `UserCreated`, `DocumentUploaded`) rather than calling each other directly, preventing cascading timeouts.

### C. Data Strategy: Database-per-Service

To ensure strict boundary contexts, no two microservices share a database.

- **Identity Service:** PostgreSQL (Relational mapping for users/roles).
- **Data Ingestion Service:** MongoDB (Flexible schema for raw payloads).
- **AI Intelligence Service:** Pinecone / Milvus (Vector embeddings).

### D. AI Integration: Agentic Orchestration

- **Orchestrator:** A dedicated **FastAPI** service powered by **LangChain & LangGraph**.
- **Behavior:** Instead of embedding AI sequentially into Node.js, the AI service operates as an asynchronous agent. It listens for system events, executes cyclic reasoning (LangGraph), accesses vector storage, and emits a completion event back to the broker.

### E. Caching Strategy: Multi-Tiered

- **Edge Caching:** Next.js caches static pages and platform shells (CDN level).
- **Gateway Rate Limiting:** The API Gateway uses **Redis** to track IP/Token request rates.
- **Data Caching:** The BFF and internal microservices use **Redis** to cache expensive database queries and AI generation results, minimizing latency and compute costs.

### F. Observability & Alerting (The Telemetry Layer)

- **Distributed Tracing:** **OpenTelemetry** is injected into every service, attaching a unique `trace_id` to every request from the Next.js client all the way down to the AI vector search.
- **Metrics & Monitoring:** Services expose `/metrics` endpoints scraped by **Prometheus**. **Grafana** dashboards visualize service health, queue depths, and cache hit rates.
- **Alerting:** Prometheus Alertmanager watches for anomalies (e.g., Kafka queue backing up, high 500-error rates) and routes alerts to developer channels (Slack/PagerDuty).

---

## 2. System Architecture Topology

```mermaid
flowchart TB
    %% Styling
    classDef client fill:#ffffff,stroke:#333,stroke-width:2px;
    classDef edge fill:#e0f7fa,stroke:#006064,stroke-width:2px;
    classDef service fill:#e8f5e9,stroke:#2e7d32,stroke-width:1px;
    classDef data fill:#fff3e0,stroke:#e65100,stroke-width:1px;
    classDef broker fill:#fce4ec,stroke:#880e4f,stroke-width:2px,stroke-dasharray: 5 5;
    classDef ai fill:#f3e5f5,stroke:#4a148c,stroke-width:2px;
    classDef ops fill:#eceff1,stroke:#455a64,stroke-width:2px,stroke-dasharray: 3 3;

    Client([Next.js Web Client]):::client

    subgraph Perimeter ["🛡️ Edge & API Entry"]
        direction TB
        Gateway[API Gateway<br/>(Rate Limit / Auth)]:::edge
        BFF[GraphQL BFF<br/>(Data Stitching)]:::edge
        Gateway -->|REST| BFF
    end

    subgraph Services ["⚙️ Core Microservices"]
        direction TB
        Identity[Identity & IAM Service]:::service
        Ingestion[Data Ingestion Service]:::service

        Redis[(Redis Cache)]:::data
        DB_ID[(PostgreSQL)]:::data
        DB_Ingest[(MongoDB)]:::data

        Identity --- DB_ID
        Ingestion --- DB_Ingest
        BFF -.->|Cache Hit| Redis
        Gateway -.->|Rate Limiting| Redis
    end

    subgraph EventMesh ["⚡ Event-Driven Backbone"]
        Kafka{{Kafka / RabbitMQ Broker}}:::broker
    end

    subgraph Intelligence ["🧠 AI Domain"]
        direction TB
        FastAPI[AI Orchestrator<br/>(LangGraph)]:::ai
        Vector[(Vector DB)]:::data
        FastAPI --- Vector
    end

    subgraph Ops ["🔍 Observability & Alerting"]
        Prometheus[Prometheus / OpenTelemetry]:::ops
        Grafana[Grafana Dashboards]:::ops
        Prometheus --- Grafana
    end

    %% Routing Flow
    Client ==>|HTTPS / WAF| Gateway
    BFF ==>|REST| Identity
    BFF ==>|REST| Ingestion

    %% Async Flow
    Identity -.->|Publishes Event| Kafka
    Ingestion -.->|Publishes Event| Kafka
    Kafka -.->|Consumes Event| FastAPI
    FastAPI -.->|Emits AI Result| Kafka

    %% Telemetry Flow
    Perimeter -.->|Traces & Metrics| Prometheus
    Services -.->|Traces & Metrics| Prometheus
    Intelligence -.->|Traces & Metrics| Prometheus
```
