# 🏗️ Engineering Platform: Internal Developer Portal

Welcome to the central Engineering Platform. This repository serves as the Internal Developer Portal (IDP), system catalog, and architectural source of truth for my distributed ecosystem.

---

## 🗺️ High-Level Architecture

```mermaid
flowchart TB
    %% Styling Definitions
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

---

## 📂 Domain & Capability Catalog

Our ecosystem is divided into four primary pillars. Click into any capability to view its architectural documentation, system design, and links to the deployable code repositories.

### 1. 🌐 Web & Platform Domain

_Translating core Javascript concepts up to full Next.js platform rendering._

- **Core Engine & Performance**
  - [V8 Callstack & Memory Profiling](domains/web-platform/core-performance/v8-profiling.md)
- **Core UI Logic & Patterns**
  - [Vanilla JS Architecture & State Patterns](domains/web-platform/core-logic/js-patterns.md)
- **Design Systems & Component Libraries**
  - [Enterprise UI Library (React/Storybook)](domains/web-platform/design-systems/enterprise-ui.md)
- **Platform Applications (SSR/SSG)**
  - [Platform Portal Dashboard (Next.js)](domains/web-platform/applications/platform-portal.md)
- **Experimental Prototypes**
  - [UI Sandbox & Experiments](domains/web-platform/prototypes/ui-sandbox.md)

### 2. ⚙️ Core Services Domain

_Backend microservices ranging from API gateways to data ingestion._

- **Identity & Access Management (IAM)**
  - [Auth & JWT API Gateway (Node.js)](domains/core-services/iam/auth-gateway.md)
- **Data Routing & REST/GraphQL APIs**
  - [High-Throughput Data Service (FastAPI)](domains/core-services/data-routing/ingestion-api.md)
- **Event-Driven Architecture (EDA)**
  - [Asynchronous Task Worker (RabbitMQ/Kafka)](domains/core-services/event-driven/task-worker.md)
- **Database & Caching Strategies**
  - [Redis Caching & Rate Limiting POC](domains/core-services/data-storage/caching-strategies.md)
- **Backend Prototypes & Integrations**
  - [Serverless & API Sandbox](domains/core-services/prototypes/backend-sandbox.md)

### 3. 🧠 Intelligence Domain

_AI pipelines spanning foundational ML to autonomous agentic systems._

- **Foundational ML & NLP**
  - [Text Classification & Sentiment API](domains/intelligence/foundational/nlp-service.md)
- **Generative AI & LLM Pipelines**
  - [RAG (Retrieval-Augmented Generation) Processor](domains/intelligence/generative/rag-processor.md)
- **AI System Design & Eval**
  - [Model Evaluation & Prompt Architecture](domains/intelligence/system-design/eval-framework.md)
- **Autonomous Agents & Tool Calling**
  - [Copilot Orchestration Agent](domains/intelligence/agents/copilot-agent.md)
- **AI Prototypes & Experiments**
  - [Model Sandbox & Jupyter Notebooks](domains/intelligence/prototypes/ai-sandbox.md)

### 4. 🏛️ Architecture & System Design

_High-level system blueprints mapping directly to the enterprise concepts._

- **Frontend System Design**
  - [Micro-Frontends & Client-Side Scaling](architecture/frontend-design/scaling-strategies.md)
- **Backend System Design**
  - [Distributed Patterns & CAP Theorem](architecture/backend-design/distributed-patterns.md)
- **Architectural Core Concepts**
  - [Data Modeling & Service Boundaries](architecture/core-concepts/data-modeling.md)
- **Enterprise Strategy**
  - [Platform Vision & Tech Radar](architecture/enterprise-strategy/platform-vision.md)
- **Architecture Sandbox & ADRs**
  - [Architecture Decision Records (ADRs)](architecture/prototypes/adrs-and-drafts.md)

---

## 🛠️ Engineering Standards

All deployable repositories linked above adhere to the following platform standards:

1. **Containerization:** All services include a `Dockerfile`.
2. **CI/CD:** Automated testing and linting via GitHub Actions.
3. **Commit Convention:** Strict adherence to Conventional Commits (e.g., `feat:`, `fix:`, `chore:`).

---

## 🌍 Enterprise System Landscape (Code Repositories)

This platform operates on a strict microservice architecture. To ensure independent CI/CD lifecycles, strict decoupling, and domain isolation, the code is split across the following dedicated repositories:

- 🏢 **[@abhishekchaturvedi07/](https://github.com/abhishekchaturvedi07)** _(GitHub Organization / User)_
  - 🏗️ [**engineering-platform**](https://github.com/abhishekchaturvedi07/engineering-platform) — _(You are here)_ Architecture, System Design, and IDP Docs.
  - 💻 [**platform-portal-app**](https://github.com/abhishekchaturvedi07/platform-portal-app) — Next.js Frontend UI & GraphQL BFF layer.
  - 🔐 [**identity-service**](https://github.com/abhishekchaturvedi07/identity-service) — Node.js IAM, JWT Auth, and PostgreSQL database.
  - 🧠 [**ai-orchestrator**](https://github.com/abhishekchaturvedi07/ai-orchestrator) — Python/FastAPI LangGraph Agent & Vector processing.
