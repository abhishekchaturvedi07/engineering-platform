# 🏛️ Enterprise System Blueprint & Data Flow

This document outlines the high-level architectural patterns, data flow, and strict domain boundaries for the platform. It is designed for enterprise scale, incorporating resilience patterns, CQRS, and strict security perimeters.

## 1. Core Architectural Decisions

### A. Edge & API Layer

- **CDN & WAF:** Cloudflare/AWS CloudFront handles static asset caching and Web Application Firewall (WAF) rules before traffic ever hits the servers.
- **API Gateway:** A dedicated layer for SSL termination, IP filtering, and OAuth2/JWT validation.
- **GraphQL BFF:** Sits securely behind the gateway, stitching together data for the Next.js frontend to prevent over-fetching.

### B. Microservice Communication & Resilience

- **External (Sync):** Client to Gateway/BFF uses REST and GraphQL over HTTPS.
- **Internal (Sync):** Service-to-Service synchronous communication utilizes **gRPC** with Protocol Buffers for high-performance, strongly-typed contracts.
- **Resilience Patterns:** All synchronous internal calls are wrapped in **Circuit Breakers** to prevent cascading failures if a downstream service degrades.
- **Internal (Async):** A Kafka/RabbitMQ event broker handles decoupled workflows. Includes a **Dead Letter Queue (DLQ)** for failed message processing.

### C. Data Strategy & CQRS

- **Database-per-Service:** No shared databases. Identity uses PostgreSQL; Ingestion uses MongoDB.
- **CQRS (Command Query Responsibility Segregation):** Write operations (Commands) are processed and stored in primary databases. Read operations (Queries) are served primarily from **Redis**, which sits aggressively in front of all databases to handle high-throughput reads.
- **Secrets Management:** HashiCorp Vault / AWS Secrets Manager centrally manages all DB credentials and AI API keys.

### D. AI Integration: Agentic Orchestration

- **Orchestrator:** A FastAPI service powered by **LangGraph** operates as an asynchronous agent.
- **Guardrails:** Inputs and outputs pass through a security layer to check for PII and hallucination bounding before interacting with the Pinecone Vector DB.

### E. DevOps & Observability

- **CI/CD:** Automated pipelines (GitHub Actions) handle testing, containerization (Docker), and deployment.
- **Telemetry:** OpenTelemetry injects trace IDs. Logs are aggregated via ELK/Datadog, and metrics are scraped by Prometheus/Grafana.

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
    classDef sec fill:#ffebee,stroke:#b71c1c,stroke-width:2px;
    classDef ops fill:#eceff1,stroke:#455a64,stroke-width:2px,stroke-dasharray: 3 3;

    Client([Clients: Web / Mobile]):::client

    subgraph EdgeLayer ["🌐 Edge & Security Perimeter"]
        direction TB
        CDN["CDN & WAF"]:::edge
        Gateway["API Gateway<br/>(Auth / Rate Limits)"]:::edge
        BFF["GraphQL BFF<br/>(Data Stitching)"]:::edge
        CDN --> Gateway --> BFF
    end

    subgraph Services ["⚙️ Core Microservices (CQRS)"]
        direction TB
        Identity["Identity Service"]:::service
        Ingestion["Data Ingestion Service"]:::service

        Cache[(Redis Read Cache)]:::data
        DB_ID[(PostgreSQL - Writes)]:::data
        DB_Ingest[(MongoDB - Writes)]:::data

        BFF -.->|Query Hit| Cache
        Identity --- DB_ID
        Ingestion --- DB_Ingest
        Identity -.->|Cache Update| Cache
        Ingestion -.->|Cache Update| Cache
    end

    subgraph Infrastructure ["🔒 Security & Ops"]
        direction LR
        Vault["Secrets Manager<br/>(Vault)"]:::sec
        CICD["CI/CD Pipeline<br/>(Docker / GitOps)"]:::ops
    end

    subgraph EventMesh ["⚡ Event-Driven Backbone"]
        direction LR
        Kafka{{"Event Broker<br/>(Kafka/RabbitMQ)"}}:::broker
        DLQ[["Dead Letter Queue (DLQ)"]]:::broker
        Kafka -.->|Failed Events| DLQ
    end

    subgraph Intelligence ["🧠 AI Domain"]
        direction TB
        FastAPI["AI Orchestrator<br/>(LangGraph)"]:::ai
        Vector[("Vector DB")]:::data
        FastAPI --- Vector
    end

    subgraph Telemetry ["🔍 Observability"]
        direction LR
        Otel["OpenTelemetry<br/>(Tracing)"]:::ops
        Grafana["Prometheus / Grafana"]:::ops
    end

    %% Routing Flow
    Client ==>|HTTPS| CDN
    BFF ==>|gRPC + Circuit Breaker| Identity
    BFF ==>|gRPC + Circuit Breaker| Ingestion

    %% Async Flow
    Identity -.->|Publish Event| Kafka
    Ingestion -.->|Publish Event| Kafka
    Kafka -.->|Consume Event| FastAPI
    FastAPI -.->|Emit Result| Kafka

    %% Global Connections
    Services -.->|Fetch Keys| Vault
    Intelligence -.->|Fetch Keys| Vault
    Services -.->|Traces| Otel
    Intelligence -.->|Traces| Otel
```
