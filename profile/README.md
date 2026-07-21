<h1 align="center">TaskForce</h1>

<p align="center">
  <b>Describe the outcome. TaskForce orchestrates the execution.</b><br/>
  <sub>The execution layer that sits over the tools your team already uses — it understands the work, drives the tools, and helps you decide.</sub>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-beta-f59e0b?style=flat-square" alt="Beta" />
  <img src="https://img.shields.io/badge/backend-Java%2021%20·%20Spring%20Boot%204-111827?style=flat-square" alt="Backend" />
  <img src="https://img.shields.io/badge/frontend-Next.js%2016%20·%20React%2019-111827?style=flat-square" alt="Frontend" />
  <img src="https://img.shields.io/badge/infra-Docker%20·%20Keycloak%20·%20PostgreSQL-111827?style=flat-square" alt="Infra" />
</p>

<p align="center">
  <img src="./assets/brain-os-graph.png" alt="TaskForce — Brain OS knowledge graph" width="90%" />
</p>

---

## What it is

Project tools **track** work. TaskForce is the layer that **understands** it.

Teams already have Linear, Notion, GitHub, Claude — they work, and nobody switches.
Project management is a solved, crowded market; the value isn't in rebuilding it. The gap
is that **nothing understands the work across those tools** — what a project is really
about, where the expertise lives, which action moves it forward, and what a decision will
cost two steps down the line.

TaskForce sits on top of the tools a team already uses and:

- **orchestrates the flow** between them and the AI that acts on them — Linear ↔ Claude —
  so no one has to leave their tools;
- **understands the stakes** of each project and **concentrates its context and expertise**
  into one model;
- **drives the tools** that move work forward, and **supports decisions** — down to
  predicting the consequences of a decision before it is made.

That intelligence core is **Brain OS**: a persistent, navigable model of a project — its
knowledge, its history, the consequences of past actions — that a human and an agent can
reason over alike. Not a project-management clone, not an ERP: **the layer that turns
intent into shipped outcomes.**

> **Status — beta, and evolving.** What runs end-to-end today is a working, dockerized
> workspace (not yet publicly hosted — the screenshots below are real). The orchestration
> and decision-intelligence layer described here is the direction Brain OS is driving
> toward, not a shipped feature. Capability, never traction.

---

## What ships today

| | Capability | What it does |
| --- | --- | --- |
| 🧭 | **One workspace** | Projects, cycles, issues, backlog, roadmap, pages and analytics in one place — board, list and roadmap views. |
| 🤖 | **Smart Assign** | Recommends the right owner for each open issue by skill, current workload and history — assignment becomes one click instead of a planning meeting. |
| 📊 | **AI Insights** | An executive read on throughput, capacity and what is at risk, generated without anyone building a report. |
| 💬 | **Ask AI** | A contextual assistant that answers from the team's real workspace, not a generic chat. |
| 🧠 | **Brain OS** | The workspace's knowledge as a navigable graph — readable by a human and by an agent alike. |

**On the roadmap (framed as direction, not shipped):** autonomous multi-agent execution
(*Agents*), real-time messaging (*Messages*, *Discussions*).

<p align="center">
  <img src="./assets/dashboard.png" alt="TaskForce dashboard" width="49%" />
  <img src="./assets/smart-assign.png" alt="TaskForce Smart Assign" width="49%" />
</p>

---

## System architecture

A multi-tenant monorepo. Identity is delegated to Keycloak; the frontend talks to the API
over REST with a bearer JWT for commands and over **WebSocket/STOMP** for anything
real-time, with RabbitMQ acting as the STOMP relay. AI runs **inside the Java backend**
(no separate inference service), and every request is traced end-to-end through
OpenTelemetry.

```mermaid
flowchart TB
    U(["Browser"])

    subgraph APP ["Application"]
        LAND["Landing<br/>Astro 5"]
        FE["Frontend<br/>Next.js 16 · React 19 · TS"]
        API["Backend API<br/>Spring Boot 4 · Java 21<br/>Clean Architecture · multi-tenant"]
    end

    NGX["Nginx · TLS<br/>reverse proxy (prod)"]
    KC["Keycloak<br/>OIDC · JWT · RBAC"]

    subgraph DATA ["Data & state"]
        PG[("PostgreSQL 18<br/>+ pgvector")]
        RDS["Redis<br/>rate limiting"]
        S3[("MinIO · S3")]
        RMQ["RabbitMQ<br/>STOMP relay"]
    end

    subgraph EXT ["Third parties"]
        GROQ["Groq · LLM"]
        STRIPE["Stripe"]
        OAUTH["GitHub / Slack"]
        SMTP["SMTP"]
    end

    OBS["SigNoz<br/>traces · metrics · logs"]

    U -->|HTTPS| LAND
    U -->|HTTPS| NGX
    NGX --> FE
    FE -->|"REST /api/** · Bearer JWT"| API
    FE <-->|"WebSocket · STOMP"| API
    FE -->|OIDC login| KC

    API -->|"OIDC · admin"| KC
    API -->|JDBC| PG
    API --> RDS
    API -->|"S3 objects"| S3
    API <-->|STOMP| RMQ
    API -->|LLM| GROQ
    API -->|"payments · webhooks"| STRIPE
    API -->|OAuth| OAUTH
    API -->|email| SMTP
    API -. OTLP .-> OBS

    style API fill:#111827,color:#fff
    style FE fill:#0ea5e9,color:#fff
    style KC fill:#7c3aed,color:#fff
```

### Stack

| Layer | Technology |
| --- | --- |
| **Backend** | Java 21, Spring Boot 4, Clean Architecture, multi-tenant, REST + WebSocket/STOMP |
| **Frontend** | Next.js 16, React 19, TypeScript, Tailwind CSS |
| **Landing** | Astro 5 |
| **AI** | In-backend LLM orchestration (Groq), MCP server, pgvector embeddings |
| **Identity** | Keycloak — OIDC, JWT, RBAC |
| **Data** | PostgreSQL 18 + pgvector, Redis, MinIO (S3), RabbitMQ (STOMP relay) |
| **Third parties** | Stripe, GitHub / Slack OAuth, SMTP |
| **Observability** | OpenTelemetry → SigNoz (ClickHouse) |
| **Security** | OWASP ZAP (DAST), Trivy & Semgrep (SAST), distributed rate limiting |
| **Delivery** | Docker Compose (dev / prod / tools), GitHub Actions, GHCR, Nginx, Render |

---

## Delivery pipeline

Every pull request runs the full quality gate — including **end-to-end tests against a
real Docker stack**, not mocks.

```mermaid
flowchart LR
    PR["Commit · Pull request"] --> CI{{"GitHub Actions"}}

    CI --> BT["Backend<br/>JUnit · JaCoCo"]
    CI --> FT["Frontend<br/>ESLint · Jest"]
    CI --> E2E["E2E · Playwright<br/>full Docker stack + seed"]
    CI --> SEC["Security<br/>Trivy · Semgrep · OWASP ZAP"]

    BT --> COV["Codecov"]
    BT --> REL
    FT --> REL
    E2E --> REL
    SEC --> REL["Release<br/>versioning · changelog"]

    REL --> IMG["GHCR<br/>container images"]
    IMG --> PROD["docker compose prod<br/>Render"]

    style CI fill:#111827,color:#fff
    style SEC fill:#b91c1c,color:#fff
    style PROD fill:#16a34a,color:#fff
```

---

## How the project is run

Three repositories, one loop: the product generates reality, the knowledge base captures
and challenges it, and the studio turns it into narrative.

```mermaid
flowchart LR
    CODE["<b>taskforce-fullstack</b><br/>product · code · CI · infra"]
    DOCS["<b>taskforce-docs</b><br/>Brain OS · architecture<br/>ADR · runbooks · R&D"]
    MOTION["<b>taskforce-motion</b><br/>studio · capture → strategy<br/>→ narrative → Remotion → channels"]

    CODE -->|"product reality"| DOCS
    DOCS -->|"decisions · contracts · specs"| CODE
    CODE -->|"screenshots · flows"| MOTION
    DOCS -->|"positioning · ICP"| MOTION

    style CODE fill:#111827,color:#fff
    style DOCS fill:#0ea5e9,color:#fff
    style MOTION fill:#7c3aed,color:#fff
```

---

## Repositories

| Repo | What's inside |
| --- | --- |
| **[taskforce-docs](https://github.com/taskforce-project/taskforce-docs)** · public | The **Brain OS** documentation vault — architecture (incl. C4 and UML), ADRs, API contracts, data model, runbooks, security and R&D. An engineering knowledge base, versioned as an Obsidian vault. |
| **taskforce-fullstack** · 🔒 private | The product monorepo. The code is kept private; its architecture is documented above and in the docs vault. |
| **taskforce-motion** · 🔒 private | **TaskForce Studio** — a marketing and motion OS, not a Remotion project: product capture feeds strategy, strategy drives narrative, Remotion is only the execution engine. No asset without a marketing hypothesis. |

How the monorepo is organized:

```
taskforce-fullstack/
├─ backend/          Spring Boot API (Java 21, Clean Architecture)
├─ frontend/         Next.js 16 app (React 19, TypeScript)
├─ landing-page/     Astro marketing site
├─ taskforce-mcp/    Model Context Protocol server
├─ keycloak/         Identity & access management
├─ nginx/            Reverse proxy / TLS gateway
├─ observability/    OpenTelemetry + SigNoz
├─ rabbitmq/         STOMP relay for realtime
├─ security-reports/ OWASP ZAP / Trivy / Semgrep
├─ scripts/          Tooling & automation
└─ docker-compose.{dev,prod,tools}.yml
```

> **Brain OS (R&D)** — the research arm behind TaskForce: a persistent-memory architecture
> for LLM agents (knowledge graph + vector search) and a *world model* that represents the
> consequences of an agent's actions.

---

## Engineering approach

- **Clean Architecture** and a multi-tenant domain model from the start.
- **Documentation as a system**, not an afterthought — an AI-assisted, human-reviewed
  knowledge vault where every technical claim is traced back to the code, and divergence
  between docs and code is flagged rather than hidden.
- **Security by default** — OIDC/JWT via Keycloak, RBAC, distributed rate limiting, and
  automated DAST/SAST scanning (OWASP ZAP, Trivy, Semgrep).
- **Observability built in** — OpenTelemetry traces, metrics and logs, collected in SigNoz.
- **Tested against reality** — E2E runs Playwright against the full dockerized stack.
- **Reproducible delivery** — the whole stack comes up with a single `docker compose`.

---

<p align="center">
  <sub>Built by <a href="https://github.com/Miche1-Pierre">Pierre Michel</a> · fil-rouge of the RNCP level-6 title, Metz Numeric School · 2025–2026</sub>
</p>
