# 🏗️ Engineering Platform: Internal Developer Portal

Welcome to the central Engineering Platform. This repository serves as the Internal Developer Portal (IDP), system catalog, and architectural source of truth for my distributed ecosystem.

---

## 🗺️ High-Level Architecture

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
