<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 2.0.0
Type of bump: MAJOR (replacing locked stack component: Agno → LangChain Deep Agents)
Date amended: 2026-03-15

Modified principles:
  - Principle VI (Confirmed Stack Non-Negotiability):
    "Replace Agno with another agent framework" →
    "Replace LangChain Deep Agents with another agent framework"

Modified sections:
  - Architecture Overview: Agno orchestration → LangChain Deep Agents
  - Component Specifications: "Agno Agent Framework" →
    "LangChain Deep Agents Framework"
  - Agent Lifecycle (Spawning, Execution): updated API references
  - Data Flow Diagrams: updated agent instantiation description
  - Observability Contract: AgnoInstrumentor →
    LangChainInstrumentor

Added sections: N/A

Removed sections: N/A

Templates requiring updates:
  ✅ .specify/templates/plan-template.md
     — "Constitution Check" section present and compatible
  ✅ .specify/templates/spec-template.md
     — Requirements section compatible with constitution constraints
  ✅ .specify/templates/tasks-template.md
     — Phase structure supports observability/security task types
  ✅ .specify/templates/agent-file-template.md
     — Generic template; no constitution-specific references needed
  ✅ CLAUDE.md — Locked Stack table updated

Follow-up TODOs:
  - TODO(NATS_RETENTION): Define JetStream stream retention policy
    for audit/replay
  - TODO(GRAPHITI_VERSION): Confirm Graphiti version and
    LangChain Deep Agents integration path
  - TODO(COPILOTKIT_AUTH): Confirm CopilotKit AG-UI auth token
    forwarding strategy
  - TODO(CANARY_POLICY): Define canary rollout percentages per service
  - TODO(OLLAMA_DEPLOYMENT): Confirm Ollama K8s deployment model
    (GPU node pool vs external)
  - TODO(TENANT_OFFBOARDING): Define data retention/deletion for
    churned tenants (GDPR implications)
-->

# OpenFactory Constitution

## Core Principles

### I. Async-First, Event-Driven Architecture (NON-NEGOTIABLE)

All API endpoints, task handlers, agent executors, and background
workers MUST use `async`/`await` throughout. No synchronous blocking
calls are permitted in request-handling or agent-execution paths.

- Agent tasks dispatched via Taskiq + NATS JetStream are the
  canonical pattern for long-running work (up to 10 minutes)
- Synchronous code is permitted ONLY in utility scripts, CLI
  tooling, and CPU-bound operations isolated to separate worker
  processes
- `requests`, `time.sleep()`, synchronous DB calls, and any
  blocking I/O in async context are deployment-blocking defects

**Rationale**: Enterprise agent workloads are inherently I/O-bound
and long-running. Synchronous blocking at any point in the stack
causes cascading queue saturation and unpredictable latency.

### II. Zero Static Credentials (NON-NEGOTIABLE)

No secrets, API keys, database passwords, or LLM provider
credentials may appear in source code, environment variable files
(`.env`), Docker images, Helm chart values files, or any
VCS-tracked artifact.

- All secrets MUST be fetched at runtime from OpenBao using K8s
  ServiceAccount JWT authentication
- Dynamic short-lived credentials (PostgreSQL, Redis) MUST be used
  exclusively via OpenBao lease mechanism
- Static long-lived credentials anywhere in the deployment pipeline
  are a deployment-blocking defect
- Local development: `.env.local` (git-ignored) is the ONLY
  permitted exception

**Rationale**: OpenFactory handles enterprise workflows across CRM,
email, and financial tooling. A single leaked credential in a
multi-tenant deployment is a systemic breach, not an isolated
incident.

### III. Tenant Isolation at Every Layer (NON-NEGOTIABLE)

Every data access pattern MUST enforce tenant boundaries:

- **PostgreSQL**: Row-level security (RLS) policies keyed on
  `tenant_id`
- **Qdrant**: One collection namespace per tenant per agent type
- **Mem0**: All memory operations scoped with `user_id` +
  `agent_id` + `tenant_id`
- **OpenBao**: One namespace per tenant; cross-namespace access
  is forbidden
- **Zitadel**: One organization per enterprise client; token
  claims carry `org_id`
- **Redis**: Key prefix `{tenant_id}:` on all cached values
- **NATS**: Subject prefix `tasks.{tenant_id}.` for task routing

Cross-tenant data access, even read-only, is a critical security
defect. MUST be caught in code review and integration tests before
any feature merges.

**Rationale**: Multi-tenant B2B SaaS. A tenant data leak is a
contractual and regulatory violation.

### IV. Observability as a First-Class Concern (NON-NEGOTIABLE)

Every agent execution, LLM invocation, API request, and background
task MUST emit:

- A structured log record (Loguru JSON) containing `trace_id`,
  `span_id`, `tenant_id`, `agent_id`
- An OpenTelemetry span with appropriate routing:
  - LLM spans + agent traces → Langfuse
  - Infrastructure traces → Grafana Tempo
- A Prometheus metric increment at minimum (request count,
  duration histogram)

Features MUST NOT ship without corresponding observability
instrumentation. The Observability Contract (Section 9) defines
the minimum required signals per component.

**Rationale**: Enterprise clients require SLA accountability.
Debugging distributed agent workflows without correlated traces
is operationally impossible at scale.

### V. Agent Factory Pattern (NON-NEGOTIABLE)

All agent orchestration MUST follow the factory dispatcher pattern:

- A single entry-point dispatcher agent receives requests and
  routes to or dynamically spawns specialized agents
- Specialized agents MUST NOT be called directly from API
  handlers — all agent invocation MUST go through the factory
  dispatcher or an authorized sub-dispatcher
- Agent types are registered in the agent registry (PostgreSQL +
  Redis cache)
- The factory pattern is the enforcement boundary for tenant
  isolation, observability injection, and secret provisioning

**Rationale**: Direct agent calls from API handlers bypass tenant
validation, observability injection, and secret lifecycle
management. The factory is the contract, not a convenience.

### VI. Confirmed Stack Non-Negotiability

The technology stack defined in this constitution is confirmed
and NON-NEGOTIABLE. Engineers MUST NOT:

- Substitute alternative ORMs for SQLAlchemy
- Use alternative task queues for Taskiq + NATS JetStream
- Replace LangChain Deep Agents with another agent framework
- Use alternative vector stores in lieu of Qdrant
- Use alternative logging libraries in lieu of Loguru
- Use alternative secrets backends in lieu of OpenBao
- Use alternative IAM in lieu of Zitadel

Deviations require a formal constitution amendment with
architectural justification, team review, and a migration plan.
Unapproved substitutions are a blocking code review defect.

**Rationale**: Stack consistency is essential for a viable
open-source project. Fragmentation creates onboarding friction
and security review surface.

### VII. Document-First Knowledge Ingestion

All knowledge ingestion into agent RAG layers MUST go through
Docling's parsing pipeline with HybridChunker before insertion
into Qdrant.

- Raw text insertion bypassing Docling's structural hierarchy
  preservation is forbidden for supported document formats
- Supported formats: PDF, DOCX, PPTX, XLSX, HTML, images, audio,
  LaTeX, CSV, AsciiDoc
- The DoclingDocument intermediate format MUST be preserved during
  ingestion for auditability and re-ingestion

**Rationale**: Structure-aware chunking is critical for accurate
RAG retrieval in enterprise documents. Naive text splitting loses
table structure, section hierarchy, and cross-reference context.

### VIII. Multi-Provider LLM Routing with Graceful Fallback

The system MUST support all five LLM providers simultaneously:
Azure OpenAI, OpenAI, Anthropic, Google Gemini, and Ollama.

- Provider selection MUST be configurable per-request and
  per-agent-type
- The routing layer MUST implement graceful fallback when a
  provider is unavailable or returns a rate-limit error
- Provider credentials are fetched from OpenBao at agent
  instantiation time
- Hardcoding a provider or assuming a single provider is
  available is a defect

**Rationale**: Enterprise clients have varying compliance
requirements. Some require Azure OpenAI for data residency;
others require Ollama for air-gapped deployments.

### IX. Simplicity and YAGNI

Complexity MUST be justified against a concrete, current
requirement.

- Abstractions created for hypothetical future use are defects
- Over-engineered service boundaries and premature generalization
  are defects
- The minimum viable implementation for the current requirement
  is the correct implementation
- When a simpler approach exists that satisfies the current
  requirement, it MUST be chosen over the more complex approach
- Complexity introduced without documented justification in the
  implementation plan MUST be challenged in code review

**Rationale**: OpenFactory is complex by necessity (multi-tenancy,
async, distributed agents). Unnecessary complexity compounds
existing necessary complexity into an unmaintainable codebase.

### X. Test-Driven for Agent Contracts

Agent input/output contracts (request schemas, response schemas,
tool call signatures) MUST have contract tests written and failing
before implementation.

- Unit tests are recommended but optional
- Integration tests for agent workflows spanning multiple services
  are REQUIRED for all P1 user stories
- The Red-Green-Refactor cycle is enforced for contract and
  integration tests
- LLM response content is non-deterministic and MUST NOT be
  asserted on directly; assert on contract shape, status codes,
  and observable side effects

**Rationale**: Agent behavior is non-deterministic at the LLM
level but must be deterministic at the contract level. Contract
tests are the only reliable regression guard for agent
orchestration changes.

## Project Summary

OpenFactory is an open-source AI Agent Factory Orchestrator
designed to be the enterprise standard for dynamic agent spawning,
orchestration, and lifecycle management across business workflows.

**Primary Goals**:

- Provide a production-ready, multi-tenant platform for deploying
  specialized AI agents
- Enable enterprise workflow automation across CRM, CDP, email,
  productivity, and coding tooling
- Deliver a fully auditable, zero-static-credential,
  observability-first agent runtime
- Be the open-source foundation that enterprises can self-host
  or extend

**Intended Users**:

- Platform engineers deploying AI agent infrastructure for
  enterprise clients
- Developer teams building specialized agents on top of the
  OpenFactory runtime
- Enterprise clients consuming agent capabilities via API or
  CopilotKit UI
- Contributors extending the open-source factory with new agent
  types and integrations

**Non-Goals** (explicitly out of scope for v1):

- Being a general-purpose LLM chat interface
- Replacing existing CRM/CDP/email SaaS platforms
- Consumer-facing deployment (designed exclusively for B2B
  enterprise)

## Architecture Overview

OpenFactory is organized into five layers, each with defined
boundaries and communication contracts.

```text
┌─────────────────────────────────────────────────────────────┐
│  UI LAYER                                                   │
│  CopilotKit (React/Next.js) ←→ AG-UI Protocol ←→ FastAPI   │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTPS / WebSocket
┌────────────────────────────▼────────────────────────────────┐
│  APPLICATION LAYER                                          │
│  FastAPI (async) → Zitadel OIDC validation → Router         │
│  SQLAlchemy ORM → PostgreSQL (CloudNativePG)                │
│  Pydantic (request/response validation)                     │
│  Loguru JSON → OpenTelemetry pipeline                       │
└────────────────────────────┬────────────────────────────────┘
                             │ Taskiq + NATS JetStream
┌────────────────────────────▼────────────────────────────────┐
│  AGENT LAYER                                                │
│  Factory Dispatcher Agent                                   │
│    ├── LangChain Deep Agents subagent spawning              │
│    ├── LangChain retriever → Qdrant (Agentic RAG)          │
│    ├── Mem0 (long-term semantic memory)                     │
│    ├── Graphiti (temporal/relational — CRM agents only)     │
│    ├── LangGraph Memory Store (short-term session memory)   │
│    └── MCP server integrations                              │
│  Specialized Agents: Email, CRM, CDP, Coding, Research,    │
│    Productivity                                             │
│  LLM Router → Azure OpenAI / OpenAI / Anthropic /          │
│    Google Gemini / Ollama                                   │
│  Docling pipeline → HybridChunker → Qdrant                 │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│  INFRASTRUCTURE LAYER                                       │
│  PostgreSQL (CloudNativePG) — tenant data, agent registry   │
│  Redis — cache, Taskiq result backend                       │
│  Qdrant — vector store (tenant-isolated collections)        │
│  NATS JetStream — task broker                               │
│  OpenBao — secrets (namespace-per-tenant, dynamic creds)    │
│  Zitadel — IAM (org-per-tenant, OIDC, JWT service accts)   │
└────────────────────────────┬────────────────────────────────┘
                             │ OTel pipeline
┌────────────────────────────▼────────────────────────────────┐
│  OBSERVABILITY LAYER                                        │
│  Langfuse — LLM traces, cost, evals (self-hosted)          │
│  Grafana Alloy — OTel collector (DaemonSet)                 │
│  Prometheus → Grafana (metrics)                             │
│  Loki → Grafana (logs, log-to-trace correlation)            │
│  Grafana Tempo (infra traces)                               │
└─────────────────────────────────────────────────────────────┘
```

**Cross-cutting concerns** (apply to all layers):

- All inter-service calls carry OTel context headers
  (`traceparent`, `tracestate`)
- All service-to-service calls carry Zitadel JWT (service account
  or user token)
- All secrets access goes through OpenBao — never environment
  variables in production
- All tenant-scoped operations validate `tenant_id` from JWT
  claims before execution

## Component Specifications

### FastAPI Application Server

**Responsibility**: HTTP API gateway, request validation, Zitadel
token validation, routing to agent dispatch or direct data
operations.

**Exposes**:

- `POST /v1/agents/dispatch` — Submit agent task (returns task_id)
- `GET /v1/agents/tasks/{task_id}` — Poll task status and result
- `GET /v1/agents/stream/{task_id}` — SSE stream for AG-UI
- `POST /v1/knowledge/ingest` — Trigger document ingestion
- `GET /v1/health` — Liveness/readiness probe
- Auto-generated OpenAPI spec at `/docs` and `/openapi.json`

**Consumes**: PostgreSQL (SQLAlchemy), Redis (cache), Taskiq
(task dispatch), Zitadel (token introspection), OpenBao (secrets).

**Configuration**: All config via OpenBao-injected environment at
pod startup. No `.env` files in production. Local dev only:
`.env.local` (git-ignored).

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| OpenBao unavailable at startup | Pod fails to start |
| Zitadel unavailable | Auth endpoints return 503; health stays 200 |
| PostgreSQL unavailable | 503 with retry-after header |
| Redis unavailable | Degrade gracefully (disable cache) |

**Interface contract** (representative Pydantic models):

```python
class AgentDispatchRequest(BaseModel):
    tenant_id: str
    agent_type: AgentType
    payload: dict[str, Any]
    provider_preference: LLMProvider | None = None
    priority: int = Field(default=5, ge=1, le=10)

class AgentDispatchResponse(BaseModel):
    task_id: str
    estimated_duration_seconds: int | None
    status: TaskStatus
```

---

### LangChain Deep Agents Framework

**Responsibility**: Agent construction, multi-agent subagent
spawning, agentic RAG (knowledge retrieval at runtime), MCP
server tool integration, session memory management via LangGraph.

**Exposes**: Agent instances consumed by factory dispatcher;
`agent.ainvoke()` async execution via LangGraph compiled graph;
LangChain retriever tools for RAG.

**Consumes**: Qdrant (via `QdrantVectorStore` + LangChain
retriever), Mem0 (wrapped as LangChain tools), Graphiti (CRM
agents), LLM provider clients (via LangChain chat model
abstraction), MCP server endpoints, LangGraph Memory Store
(short-term session state).

**Configuration**:

- Model provider configured per agent type via agent registry
  (fetched from OpenBao at dispatch time)
- Knowledge collection names follow convention:
  `{tenant_id}_{agent_type}_{collection_name}`
- Session memory TTL: configurable per agent type (default 1 hour)
- Agent created with `create_deep_agent(tools=[...], system_prompt=...)`
  from the `deepagents` package (built on LangGraph)
- LangGraph thread/memory store keyed on `{tenant_id}:{task_id}`
  for session isolation

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| LLM provider unavailable | Attempt fallback per routing policy; if all fail, 503 |
| Qdrant unavailable | Run without RAG; log `rag_degraded=true` |
| Mem0 unavailable | Run without long-term memory; log `memory_degraded=true` |
| MCP server unavailable | Specific tool fails; agent continues without it |

---

### Taskiq + NATS JetStream Task Queue

**Responsibility**: Async task dispatch, durable message delivery,
result storage, long-running job support (up to 10 minutes),
retry handling.

**Exposes**: `broker.task()` decorator for task registration;
`task.kiq()` for async dispatch; result polling via Redis.

**Consumes**: NATS JetStream (broker), Redis (result backend),
FastAPI DI (via taskiq-fastapi for shared dependency injection).

**Configuration**:

- NATS stream: `OPENFACTORY_TASKS` with subject prefix `tasks.`
- Stream retention: work-queue (messages deleted after ack)
- Consumer max delivery: 3 (configurable per task type)
- Task timeout: 600 seconds (10 minutes) per agent job
- Result TTL: 24 hours in Redis

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| NATS unavailable | Task dispatch returns 503; no silent drops |
| Redis result backend down | Results logged but not stored; use SSE |
| Task timeout | Status → `FAILED`; retry policy applied |

---

### OpenBao Secrets Management

**Responsibility**: Runtime secret delivery, dynamic credential
generation (PostgreSQL, Redis), tenant namespace isolation, K8s
ServiceAccount JWT authentication.

**Exposes**: KV secrets at `{tenant_namespace}/kv/`, dynamic DB
credentials at `{tenant_namespace}/database/creds/{role}`.

**Consumes**: K8s ServiceAccount JWT (auth), Azure Key Vault
(auto-unseal).

**Configuration**:

- Auth method: `kubernetes` with ServiceAccount JWT validation
- One namespace per tenant: `tenants/{tenant_id}/`
- PostgreSQL dynamic role: `openfactory-{tenant_id}-rw` (rw)
  and `-ro` (read-only)
- Lease TTL: 1 hour; max TTL: 24 hours; auto-renewal via sidecar

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| OpenBao sealed | Services fail to start; Azure KV auto-unseal handles rolling restarts |
| Lease expiry without renewal | Creds revoked; liveness probe triggers pod restart |
| Namespace not found | Agent dispatch fails with `tenant_not_provisioned` |

---

### Zitadel IAM

**Responsibility**: OIDC/OAuth2 for human user auth, JWT service
accounts for agent-to-agent auth, OAuth2 token introspection for
MCP server auth, multi-tenant organization management.

**Exposes**: OIDC discovery endpoint, token introspection
endpoint, JWKS endpoint.

**Configuration**:

- One Zitadel organization per enterprise client
- JWT claims MUST include: `org_id` (tenant_id), `sub`
  (user/service-account ID), `roles`
- FastAPI dependency: `get_current_tenant()` extracts and
  validates `org_id` from JWT
- Service accounts: one per agent type, credentials in OpenBao

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Zitadel unavailable | 503; cached JWKS allows ~5 min grace |
| Token expired | 401; client must refresh |

---

### PostgreSQL (CloudNativePG)

**Responsibility**: Durable storage for tenant registry, agent
registry, task metadata, user data, audit logs.

**Exposes**: PostgreSQL wire protocol; accessed exclusively via
SQLAlchemy async engine.

**Consumes**: Dynamic credentials from OpenBao.

**Configuration**:

- RLS on all tenant-scoped tables:
  `tenant_id = current_setting('app.tenant_id')`
- Application MUST call
  `SET app.tenant_id = '{tenant_id}'` at session start via
  SQLAlchemy event listener
- Separate read replica for analytics/reporting queries
- Connection pool: SQLAlchemy async pool, max 20 connections
  per service replica

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Primary unavailable | CloudNativePG failover to replica (RTO < 30s) |
| Pool exhausted | Queued with overflow; 503 if overflow exceeded |

---

### Qdrant Vector Store

**Responsibility**: Vector similarity search for agent RAG layers;
tenant-isolated collections for knowledge base separation.

**Exposes**: Qdrant REST/gRPC API; consumed via LangChain's
`QdrantVectorStore` and `create_retriever_tool` abstractions.

**Configuration**:

- Collection naming:
  `{tenant_id}_{agent_type}_{collection_name}` (enforced by
  factory)
- Embedding dimensions: provider-dependent (must be consistent
  per collection)
- Payload filters on all queries MUST include `tenant_id` as a
  redundant safety filter
- Snapshot-based backup: daily

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Qdrant unavailable | Agents degrade to no-RAG; `rag_degraded=true` |
| Collection not found | Ingestion creates on first write; query returns empty |

---

### Langfuse (LLM Observability)

**Responsibility**: LLM span capture, prompt versioning, cost
tracking, multi-turn conversation tracing, LLM-as-judge evals.

**Exposes**: OTel OTLP endpoint (HTTP); Langfuse dashboard.

**Consumes**: OTel spans from LangChain Deep Agents via
`LangChainInstrumentor` + LangfuseSpanProcessor.

**Configuration**:

- `blocked_instrumentation_scopes`: `["fastapi", "sqlalchemy"]`
  — infrastructure spans MUST NOT reach Langfuse
- LangChain instrumented via `LangChainInstrumentor().instrument()`
  at app startup
- `LangfuseSpanProcessor` added to OTel `TracerProvider` with
  Langfuse OTLP exporter
- Prompt versions managed in Langfuse UI; fetched at agent
  instantiation via Langfuse SDK

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Langfuse unavailable | Spans dropped (non-blocking); no agent impact |
| Span export errors | Logged WARNING; agent execution continues |

---

### Docling Document Ingestion Pipeline

**Responsibility**: Parse enterprise documents into structured
DoclingDocument format; chunk with HybridChunker for
structure-aware RAG ingestion; insert into Qdrant.

**Exposes**: Internal async ingestion function; triggered via
`POST /v1/knowledge/ingest` API.

**Consumes**: Raw document bytes (upload), Qdrant (vector insert),
PostgreSQL (ingestion audit log).

**Configuration**:

- HybridChunker: `max_tokens=512`, `merge_peers=True` (defaults;
  tunable per tenant)
- Supported formats: PDF, DOCX, PPTX, XLSX, HTML, images, audio,
  LaTeX, CSV, AsciiDoc
- Ingestion runs as Taskiq background task (non-blocking)
- DoclingDocument JSON stored in PostgreSQL for audit/re-ingestion

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Unsupported format | `format_not_supported` error; no ingestion |
| Parse failure | Task fails; doc stored in quarantine for review |
| Qdrant insert failure | Retry 3x; on final failure, audit log updated |

---

### Mem0 (Long-term Semantic Memory)

**Responsibility**: Persistent semantic memory across agent
sessions; namespaced per user, agent, and tenant.

**Exposes**: Mem0 LangChain tool wrappers for Deep Agents; `add()`,
`search()`, `get_all()` operations.

**Configuration**:

- Memory namespace: `{tenant_id}/{user_id}/{agent_id}` — all
  three MUST be set
- Mem0 backend: Qdrant (shared cluster, isolated by namespace)
- Memory retention: configurable per tenant (default: 90 days)

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Mem0 unavailable | Agent runs without memory; `memory_degraded=true` |

---

### Graphiti (Temporal/Relational Memory)

**Responsibility**: CRM-type temporal reasoning and entity
relationship tracking for agents that need to understand changing
relationship states over time.

**Scope**: CRM agents (Salesforce, HubSpot) ONLY. Not used for
email, coding, or productivity agents.

**Exposes**: Graphiti client consumed directly by CRM agent
implementations.

**Configuration**:

- Neo4j backend (or compatible) for graph storage
- Entity types: Contact, Account, Opportunity, Interaction,
  StatusChange
- Temporal indexing: all entity state changes versioned with
  timestamps

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Graphiti unavailable | CRM agents degrade to Mem0-only; `graphiti_degraded=true` |

---

### Redis

**Responsibility**: Caching layer and Taskiq result backend.

**Exposes**: Redis protocol; consumed by FastAPI cache utilities
and Taskiq result backend.

**Configuration**:

- All keys prefixed with `{tenant_id}:` for tenant isolation
- Agent registry cache: TTL 5 minutes
- Task results: TTL 24 hours
- Sentinel mode for HA; Cluster mode at large scale

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| Redis unavailable | Caching disabled; task results fall back to DB polling |

---

### NATS JetStream

**Responsibility**: Durable message broker for Taskiq task
dispatch; guaranteed delivery with at-least-once semantics.

**Exposes**: NATS client protocol; consumed by Taskiq broker.

**Configuration**:

- Stream: `OPENFACTORY_TASKS`
- Subject hierarchy: `tasks.{tenant_id}.{agent_type}`
- Retention: work-queue (delete after ack)
- 3-node cluster for HA; TLS enabled

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| NATS cluster unavailable | Task dispatch returns 503; no silent drops |
| Message delivery failure | Retry up to max_delivery (3x) |

---

### CopilotKit + AG-UI Protocol

**Responsibility**: Real-time agent-user interaction UI layer;
streaming agent responses to React/Next.js frontend via AG-UI
protocol.

**Exposes**: AG-UI SSE event stream via FastAPI SSE endpoint;
CopilotKit state management.

**Consumes**: FastAPI `/v1/agents/stream/{task_id}` SSE endpoint.

**Configuration**:

- AG-UI events MUST include `tenant_id` and `agent_id` in
  event metadata
- Auth: Zitadel OIDC token forwarded from frontend to backend
  on every SSE connection

**Failure modes**:

| Failure | Behavior |
|---------|----------|
| SSE connection dropped | Client MUST reconnect and resume from last `event_id` |

## Data Flow Diagrams

### Flow 1: User Request → Agent Dispatch → Execution → Response

```text
User (Browser / API Client)
  │
  ▼ HTTPS POST /v1/agents/dispatch
  │  Authorization: Bearer {zitadel_jwt}
  │  Body: {tenant_id, agent_type, payload, provider_preference}
  │
FastAPI Handler
  │ 1. Validate Zitadel JWT → extract tenant_id, user_id, roles
  │ 2. Authorize: verify role permits agent_type
  │ 3. Validate request body (Pydantic)
  │ 4. Look up agent config in PostgreSQL (cached in Redis)
  │ 5. Emit OTel span: api.agent_dispatch
  │
  ▼ Taskiq broker.kiq(agent_task, ...)
NATS JetStream (subject: tasks.{tenant_id}.{agent_type})
  │
  ▼ Taskiq worker picks up task
Agent Task Worker
  │ 1. Fetch secrets from OpenBao (LLM creds, integration creds)
  │ 2. Instantiate factory dispatcher agent (LangChain Deep Agents)
  │ 3. Dispatcher analyzes request → routes to specialized agent
  │ 4. Specialized agent executes:
  │    a. Query Qdrant via LangChain retriever tool (RAG)
  │    b. Retrieve Mem0 long-term memory via tool wrapper
  │    c. CRM agents: query Graphiti temporal graph
  │    d. Call LLM provider (via LangChain chat model abstraction)
  │    e. Execute tool calls (MCP servers, integration APIs)
  │    f. Update Mem0 memory with new context
  │ 5. Emit OTel spans: agent.dispatch, agent.llm_call,
  │    agent.tool_call
  │ 6. Store result in Redis (Taskiq result backend)
  │
  ▼
FastAPI (SSE stream or poll)
  │ GET /v1/agents/tasks/{task_id} → poll Redis for result
  │ GET /v1/agents/stream/{task_id} → SSE AG-UI events
  │
  ▼
User receives result
```

### Flow 2: Document Ingestion → Docling → Qdrant

```text
Client
  │ POST /v1/knowledge/ingest
  │ Body: {tenant_id, agent_type, collection_name,
  │        document_bytes, format}
  │
FastAPI Handler
  │ 1. Validate JWT + tenant_id
  │ 2. Validate format is supported by Docling
  │ 3. Store raw document in PostgreSQL (audit record)
  │ 4. Dispatch ingestion task via Taskiq
  │
  ▼ Taskiq ingestion task
Docling Pipeline Worker
  │ 1. Docling.convert(document_bytes, format) → DoclingDocument
  │ 2. Store DoclingDocument JSON in PostgreSQL (audit)
  │ 3. HybridChunker(doc).chunk() → List[DocChunk]
  │ 4. Generate embeddings (via configured embedding model)
  │ 5. Upsert vectors into Qdrant:
  │    collection = {tenant_id}_{agent_type}_{collection_name}
  │    payload: {tenant_id, chunk_id, source_doc_id, page,
  │             section}
  │ 6. Update PostgreSQL audit record: status=completed,
  │    chunk_count=N
  │
  ▼
Agent knowledge base updated;
next agent query will find this content
```

### Flow 3: Secret Lifecycle — OpenBao Lease → Rotation

```text
Pod Startup (K8s)
  │
OpenBao Agent Sidecar (init container)
  │ 1. Authenticate to OpenBao via K8s ServiceAccount JWT
  │ 2. Fetch static secrets: LLM API keys, integration tokens
  │    → Written to in-memory tmpfs (/vault/secrets/)
  │ 3. Request dynamic PostgreSQL credential:
  │    Path: tenants/{tenant_id}/database/creds/openfactory-rw
  │    → Returns: {username, password, lease_id,
  │               lease_duration=3600s}
  │ 4. Request dynamic Redis credential (same pattern)
  │ 5. Write all credentials to tmpfs; notify app via file watch
  │
Application Process
  │ 1. Read credentials from tmpfs at startup
  │ 2. Initialize SQLAlchemy engine with dynamic DB credentials
  │ 3. Initialize Redis client with dynamic credentials
  │ 4. Start lease renewal timer (renew at 50% of lease TTL)
  │
OpenBao Agent Sidecar (background)
  │ Every ~30 minutes (at 50% TTL):
  │   Renew lease → new TTL issued
  │   If renewal fails: rotate credential entirely
  │   Write new credential → app detects file change → reconnect
  │
Pod Shutdown / Eviction
  │ Graceful: sidecar revokes leases
  │ Forced: leases expire naturally at max TTL (24h)
```

### Flow 4: Observability Pipeline — App → Collectors → Backends

```text
Application Services (FastAPI, Taskiq workers)
  │
  ├── Loguru JSON logs → stdout
  │   Fields: timestamp, level, message, trace_id, span_id,
  │           tenant_id, agent_id
  │
  ├── OTel Spans (via OpenTelemetry SDK)
  │   Two exporters configured:
  │   ├── LangfuseSpanProcessor → Langfuse OTLP endpoint
  │   │   Scope: agno.*, llm.*, agent.*
  │   │   (NOT fastapi/sqlalchemy)
  │   └── OTel OTLP Exporter → Grafana Alloy (DaemonSet)
  │       Scope: fastapi.*, sqlalchemy.*, taskiq.*
  │       (infra spans only)
  │
  └── Prometheus metrics → /metrics endpoint
      (scraped by kube-prometheus-stack)
  │
Grafana Alloy (DaemonSet on each K8s node)
  │ 1. Scrapes stdout logs from pod log files → Loki
  │ 2. Receives OTel traces (OTLP HTTP) → Grafana Tempo
  │ 3. Forwards Prometheus scrape targets → Prometheus
  │
  ├── Loki ← structured logs
  │   (queryable by trace_id, tenant_id, agent_id)
  ├── Grafana Tempo ← infra traces
  │   (correlated with logs via trace_id)
  ├── Prometheus ← metrics
  │   (dashboards in Grafana)
  └── Langfuse ← LLM spans
      (cost tracking, prompt versions, evals)
```

## Multi-Tenancy Model

### Tenant Provisioning Workflow

When a new enterprise client is onboarded, the following
resources MUST be provisioned in order. The provisioning process
is orchestrated by a Taskiq task triggered by the admin API.
Provisioning MUST be fully idempotent (re-runnable without side
effects).

```text
Step 1: Zitadel
  → Create new organization
  → Provision admin user
  → Generate service account JWTs per agent type
  → Store JWT credentials in OpenBao

Step 2: OpenBao
  → Create namespace: tenants/{tenant_id}/
  → Enable KV secret engine in namespace
  → Enable database secret engine in namespace
  → Create PostgreSQL roles scoped to tenant

Step 3: PostgreSQL
  → Create tenant record in tenants table
  → Enable RLS policies for tenant_id
  → Run tenant-specific migrations

Step 4: Qdrant
  → Collections created lazily on first ingestion
  → Naming convention enforced by factory

Step 5: Mem0
  → No pre-provisioning required
  → Namespacing enforced at runtime: {tenant_id}/...
```

### Layer-by-Layer Isolation Matrix

| Layer | Isolation Mechanism | Enforcement Point |
|-------|--------------------|--------------------|
| Authentication | Zitadel org_id in JWT claims | FastAPI JWT validation middleware |
| Secrets | OpenBao namespace per tenant | OpenBao namespace policy |
| PostgreSQL | Row-level security (tenant_id) | SQLAlchemy session event listener |
| Qdrant | Collection name prefix `{tenant_id}_` | Agent factory collection name builder |
| Mem0 | `user_id` + `agent_id` + `tenant_id` | Mem0 client wrapper |
| Graphiti | Tenant-scoped graph partition | Graphiti client wrapper |
| Task queue | NATS subject `tasks.{tenant_id}.` | Task dispatch utility |
| Redis cache | Key prefix `{tenant_id}:` | Cache utility wrapper |
| SSE streams | AG-UI events carry tenant_id | SSE endpoint validation |

### Cross-Tenant Access Prevention Rules

- All SQLAlchemy queries MUST go through `TenantSession` context
  manager that calls `SET app.tenant_id`
- Direct `Session` usage without `TenantSession` wrapper is a
  blocking code review defect
- Qdrant collection names validated against `{tenant_id}_` prefix
  at query time by the factory layer
- Mem0 operations validate namespace matches authenticated
  `tenant_id` before execution
- NATS consumer groups are scoped to tenant subject prefix;
  workers MUST NOT subscribe to wildcard `tasks.>`
- Redis key operations MUST go through `TenantCache` wrapper that
  enforces prefix; direct `redis.get()` is forbidden in
  application code

## Agent Lifecycle

### 1. Registration

Agent types are registered in the PostgreSQL `agent_registry`
table. The registry is cached in Redis with a 5-minute TTL.
Changes take effect within 5 minutes without requiring
deployment.

```python
class AgentRegistryEntry(Base):
    __tablename__ = "agent_registry"

    agent_type: Mapped[str]        # e.g., "email", "crm"
    display_name: Mapped[str]
    description: Mapped[str]
    capabilities: Mapped[list[str]]  # tool names
    memory_config: Mapped[dict]      # memory backends
    llm_providers: Mapped[list[str]] # supported providers
    is_active: Mapped[bool]
    tenant_overrides: Mapped[dict]   # per-tenant config
```

### 2. Spawning

```text
1. Factory dispatcher receives task (via Taskiq)
2. Load agent registry entry from Redis / PostgreSQL
3. Fetch agent-specific secrets from OpenBao:
   {tenant_namespace}/kv/agents/{agent_type}/
4. Instantiate Agno Agent with:
   - model: resolved from provider_preference + registry
   - knowledge: AgentKnowledge(
       qdrant_collection={tenant_id}_{agent_type}_*)
   - tools: Mem0Tools + agent-specific tools + MCP clients
   - memory: Agno session memory
   - For CRM agents only: add Graphiti client
5. Inject OTel span attributes:
   agent_type, tenant_id, task_id, provider
```

### 3. Execution

- All agent executions use Agno `arun()` (async)
- Maximum execution time: 600 seconds (enforced by Taskiq)
- Agent MAY spawn sub-agents via `Agent(team=[...])` pattern
- All sub-agent calls inherit parent OTel span context
- Tool call results logged at DEBUG level with sanitized payloads
  (PII fields masked)
- LLM responses NOT logged in full (cost and token counts only);
  full traces go to Langfuse
- Memory updates (Mem0, Graphiti) are performed after successful
  agent execution, not during

### 4. Monitoring

- Taskiq task statuses: `PENDING`, `RUNNING`, `SUCCESS`,
  `FAILED`, `TIMEOUT`
- Agent health metrics:
  - `agent_task_duration_seconds` histogram
  - `agent_task_total` counter (by agent_type, tenant_id, status)
- Langfuse: full LLM trace per agent execution;
  conversation-level and span-level cost tracking
- Alert conditions:
  - P99 task duration > 300s → warning
  - Error rate > 5% per agent_type per tenant → warning
  - Error rate > 15% per agent_type → critical

### 5. Retirement

- Set `is_active = false` in agent registry
- Registry cache invalidated immediately (Redis DEL)
- In-flight tasks complete; no forced termination
- No new tasks dispatched to retired agent type
- Agent-specific Qdrant collections retained (manual cleanup)
- Mem0 and Graphiti data retained per tenant retention policy

## Security Model

### Zero-Trust Principles

1. No service trusts any other service by default
2. Every request carries a verifiable identity token (Zitadel JWT)
3. Every secret is short-lived and dynamically provisioned
4. Network policies allow only explicitly required paths
5. All inter-service communication is mTLS (enforced at K8s
   network policy level)

### Authentication Matrix

| Actor | Authenticates With | Validated By |
|-------|-------------------|--------------||
| Human → FastAPI | Zitadel OIDC token | FastAPI JWT middleware |
| Agent worker → FastAPI | Zitadel service account JWT | FastAPI JWT middleware |
| FastAPI → PostgreSQL | OpenBao dynamic credential | PostgreSQL auth |
| FastAPI → Redis | OpenBao dynamic credential | Redis AUTH |
| FastAPI → Qdrant | OpenBao static secret (API key) | Qdrant API key |
| FastAPI → Zitadel | OpenBao static secret (client creds) | Zitadel OIDC |
| Agent → MCP server | OAuth2 token (Zitadel introspection) | MCP server |
| Agent → LLM provider | OpenBao static secret (API key) | LLM provider |
| K8s pod → OpenBao | ServiceAccount JWT | OpenBao K8s auth |

### Authorization Model

**Zitadel Roles**:

- `tenant_admin` — Full access to tenant config, agent
  management, knowledge ingestion
- `agent_operator` — Dispatch agents, view task results,
  ingest documents
- `agent_viewer` — View task results and agent status only

**Enforcement**:

- Role-based access enforced in FastAPI dependency:
  `require_role(role: str)`
- Agent-type permissions configurable per-role in Zitadel
- RLS in PostgreSQL enforces data access beyond role checks
- Agent dispatch validates user has permission for the
  requested `agent_type`

### Network Policies (Kubernetes)

All pods default to deny-all ingress and egress.
Explicitly allowed paths:

| Source | Destination | Port | Purpose |
|--------|-------------|------|---------|
| Ingress controller | FastAPI | 8000 | API traffic |
| FastAPI | PostgreSQL | 5432 | Database queries |
| FastAPI | Redis | 6379 | Cache + task results |
| FastAPI | Qdrant | 6333/6334 | Vector search |
| FastAPI | NATS | 4222 | Task dispatch |
| FastAPI | OpenBao | 8200 | Secret fetch |
| FastAPI | Zitadel | 443 | Token introspection |
| FastAPI | Langfuse | 3000 | Span export |
| Agent workers | LLM providers | 443 | LLM API calls |
| Agent workers | MCP servers | varies | Tool calls |
| Agent workers | Integration APIs | 443 | Salesforce, Gmail, etc. |
| All pods (labeled) | OpenBao | 8200 | Secret fetch |
| Alloy DaemonSet | Loki | 3100 | Log shipping |
| Alloy DaemonSet | Tempo | 4318 | Trace shipping |

Agent workers MUST NOT have direct internet access without an
explicit network policy entry for the target host.

### Secret Rotation Policy

| Secret Type | TTL | Max TTL | Rotation |
|-------------|-----|---------|----------|
| PostgreSQL dynamic creds | 1 hour | 24 hours | OpenBao sidecar auto-renewal |
| Redis dynamic creds | 1 hour | 24 hours | OpenBao sidecar auto-renewal |
| LLM provider API keys | 90 days | — | OpenBao KV versioning; zero-downtime via agent re-instantiation |
| Integration OAuth tokens | per provider | — | Refresh token rotation in integration layer |
| Zitadel service account JWTs | per Zitadel config | — | Stored in OpenBao; rotated via admin API |

### Security Defect Classification

| Defect | Severity | Action |
|--------|----------|--------|
| Static credential in codebase | Critical | Block merge; revoke credential immediately |
| Cross-tenant data access | Critical | Block merge; incident review |
| Missing tenant_id validation | High | Block merge |
| Bare `except Exception: pass` | High | Block merge |
| Missing OTel span on API endpoint | Medium | Must fix before release |
| Direct Session without TenantSession | High | Block merge |
| Direct Redis access without TenantCache | High | Block merge |

## Kubernetes Deployment Spec

### Helm Chart Deployment Order

Services MUST be deployed in the following dependency order.
Each step represents a Helm release. Each release MUST reach
`Ready` state before proceeding to the next.

```text
Step 1:  cert-manager
         Prerequisites: none
         Purpose: TLS certificates for all services

Step 2:  CloudNativePG operator
         Prerequisites: cert-manager
         Purpose: PostgreSQL cluster lifecycle

Step 3:  OpenBao
         Prerequisites: cert-manager, Azure Key Vault accessible
         Purpose: Bootstrap admin token; configure K8s auth;
                  create tenant namespaces; enable secret engines

Step 4:  Zitadel
         Prerequisites: CloudNativePG, OpenBao
         Purpose: IAM; requires PostgreSQL for persistence;
                  reads DB password from OpenBao

Step 5:  NATS JetStream
         Prerequisites: cert-manager
         Purpose: Task broker; configure JetStream streams; TLS

Step 6:  Redis
         Prerequisites: OpenBao
         Purpose: Cache + result backend; password from OpenBao

Step 7:  Qdrant
         Prerequisites: cert-manager
         Purpose: Vector store; API key from OpenBao

Step 8:  kube-prometheus-stack
         Prerequisites: none (can parallel with Step 7)
         Purpose: Prometheus, Alertmanager, Grafana;
                  ServiceMonitors for all components

Step 9:  Loki + Grafana Tempo
         Prerequisites: kube-prometheus-stack
         Purpose: Log aggregation and distributed tracing

Step 10: Langfuse
         Prerequisites: CloudNativePG, Redis
         Purpose: LLM observability; API keys in OpenBao

Step 11: Grafana Alloy
         Prerequisites: Loki, Tempo, Langfuse, Prometheus
         Purpose: OTel collector DaemonSet; route spans/logs

Step 12: OpenFactory API + Worker services
         Prerequisites: ALL above steps complete
         Purpose: Application deployment
```

### Resource Requirements (Per Replica, Starting Point)

| Component | CPU Req | CPU Lim | Mem Req | Mem Lim | Replicas |
|-----------|---------|---------|---------|---------|----------|
| FastAPI | 250m | 1000m | 256Mi | 1Gi | 2 min |
| Taskiq Worker | 500m | 2000m | 512Mi | 2Gi | 2 min |
| PostgreSQL primary | 500m | 2000m | 1Gi | 4Gi | 1+1 replica |
| Redis | 250m | 500m | 256Mi | 1Gi | 3 (sentinel) |
| Qdrant | 500m | 2000m | 2Gi | 8Gi | 3 (distributed) |
| NATS JetStream | 250m | 500m | 256Mi | 512Mi | 3 (cluster) |
| OpenBao | 250m | 500m | 256Mi | 512Mi | 3 (HA Raft) |
| Langfuse | 250m | 1000m | 256Mi | 1Gi | 1 dev / 2+ prod |
| Alloy DaemonSet | 100m | 250m | 128Mi | 256Mi | 1 per node |

*Starting estimates. Size to actual load via HPA metrics.*

### Scaling Strategies

- **FastAPI**: HPA on CPU + custom metric
  `http_requests_in_flight`; target 60% CPU utilization
- **Taskiq Workers**: HPA on NATS consumer pending message count
  (KEDA NATS scaler); min 2, max 20 workers
- **PostgreSQL**: Read replicas for reporting; CloudNativePG
  manages failover
- **Qdrant**: Horizontal via distributed mode; shard count
  per collection configurable
- **Redis**: Sentinel for HA; Cluster for horizontal write
  scaling at large scale
- **NATS**: Cluster mode (3 nodes minimum for JetStream HA)

### HA Considerations

| Service | HA Strategy | RTO | RPO |
|---------|-------------|-----|-----|
| OpenBao | 3-node Raft; Azure KV auto-unseal | < 30s | 0 |
| PostgreSQL | CloudNativePG auto-failover; sync replica | < 30s | 0 |
| NATS | 3-node JetStream; messages replicated 3x | < 10s | 0 |
| Qdrant | Replication factor >= 2; write quorum | < 30s | 0 |
| Redis | Sentinel with 3 nodes | < 15s | ~1s |

All stateful services MUST have `PodDisruptionBudget` with
`minAvailable: 1` during maintenance windows.

## Observability Contract

Every component MUST emit the following signals. Missing signals
from new features are a deployment-blocking defect.

### Per API Request (every FastAPI endpoint)

**OTel Span** (infra scope → Tempo):

```text
span.name = "HTTP {method} {route}"
attributes:
  http.method, http.route, http.status_code
  tenant_id, user_id (from JWT)
  http.request_content_length
```

**Prometheus Metrics**:

```text
openfactory_http_requests_total
  {method, route, status, tenant_id}
openfactory_http_request_duration_seconds
  {method, route, tenant_id}  (histogram)
```

**Log Record**:

```json
{
  "level": "INFO",
  "event": "http_request",
  "method": "POST",
  "route": "/v1/agents/dispatch",
  "status_code": 202,
  "duration_ms": 45,
  "tenant_id": "acme-corp",
  "user_id": "usr_123",
  "trace_id": "...",
  "span_id": "..."
}
```

### Per Agent Task Execution (every Taskiq agent task)

**OTel Span** (agent scope → Langfuse):

```text
span.name = "agent.{agent_type}.execute"
attributes:
  agent_type, tenant_id, task_id
  llm_provider, model_name
  rag_enabled, memory_enabled, graphiti_enabled
  token_count_input, token_count_output
  tool_calls_count
  duration_seconds
  status (success / failed / timeout)
```

**Prometheus Metrics**:

```text
openfactory_agent_tasks_total
  {agent_type, tenant_id, status}
openfactory_agent_task_duration_seconds
  {agent_type, tenant_id}  (histogram)
openfactory_agent_llm_tokens_total
  {agent_type, provider, model, type}  (input/output)
openfactory_agent_tool_calls_total
  {agent_type, tool_name, status}
```

**Log Record** (task start and end):

```json
{
  "level": "INFO",
  "event": "agent_task_complete",
  "agent_type": "crm",
  "task_id": "task_abc123",
  "tenant_id": "acme-corp",
  "duration_ms": 4500,
  "status": "success",
  "llm_provider": "azure_openai",
  "model": "gpt-4o",
  "input_tokens": 1200,
  "output_tokens": 350,
  "trace_id": "...",
  "span_id": "..."
}
```

### Per LLM Invocation (every LLM call via Agno)

**OTel Span** (LLM scope → Langfuse only):

```text
span.name = "llm.{provider}.chat"
attributes: (OpenInference semantic conventions)
  llm.provider, llm.model, llm.request.type
  llm.usage.prompt_tokens, llm.usage.completion_tokens
  llm.usage.total_tokens
  gen_ai.system, gen_ai.request.model
```

Automatically captured via AgnoInstrumentor +
LangfuseSpanProcessor. Cost calculated from token counts +
Langfuse price catalog.

### Per Background Task (non-agent Taskiq tasks)

**OTel Span** (infra scope → Tempo):

```text
span.name = "task.{task_name}"
attributes:
  task_name, tenant_id, task_id, status, duration_seconds
```

**Prometheus Metrics**:

```text
openfactory_background_tasks_total
  {task_name, tenant_id, status}
openfactory_background_task_duration_seconds
  {task_name}  (histogram)
```

### Secret Access Logging

Log record ONLY (no OTel span for security reasons):

```json
{
  "level": "DEBUG",
  "event": "secret_lease_renewed",
  "tenant_id": "acme-corp",
  "secret_path": "tenants/acme-corp/database/creds/openfactory-rw",
  "lease_duration_seconds": 3600,
  "trace_id": "..."
}
```

### Alert Thresholds (Prometheus AlertManager)

| Alert | Condition | Severity |
|-------|-----------|----------|
| AgentTaskErrorRateHigh | > 5% error per agent_type per 5m | warning |
| AgentTaskErrorRateCritical | > 15% error per agent_type per 5m | critical |
| AgentTaskDurationHigh | P99 > 300s | warning |
| AgentTaskTimeout | any status=timeout | warning |
| OpenBaoLeaseRenewalFailed | lease renewal failure log | critical |
| LLMProviderUnavailable | all fallbacks exhausted | critical |
| QdrantDegraded | rag_degraded > 10% rate | warning |
| Mem0Degraded | memory_degraded > 10% rate | warning |

## Development Guidelines

### Language and Runtime

- Python 3.12+ (enforced via `.python-version` and
  `pyproject.toml` `requires-python`)
- Package management: `uv` exclusively; do NOT use pip, poetry,
  or conda directly
- All dependencies declared in `pyproject.toml`; lock file
  (`uv.lock`) committed to VCS

### Project Structure

```text
open-factory/
├── src/
│   └── openfactory/
│       ├── api/              # FastAPI routers, deps, middleware
│       ├── agents/           # Agent definitions
│       │   ├── factory/      # Dispatcher agent
│       │   ├── email/        # Email agent
│       │   ├── crm/          # CRM agent (Graphiti config)
│       │   ├── cdp/          # CDP agent
│       │   ├── coding/       # Coding agent
│       │   ├── research/     # Research/RAG agent
│       │   └── productivity/ # Productivity agent
│       ├── knowledge/        # Docling ingestion pipeline
│       ├── memory/           # Mem0 + Graphiti wrappers
│       ├── models/           # SQLAlchemy models
│       ├── schemas/          # Pydantic request/response
│       ├── services/         # Business logic layer
│       ├── tasks/            # Taskiq task definitions
│       ├── config/           # Configuration (OpenBao-backed)
│       ├── observability/    # OTel, Loguru, metrics setup
│       └── security/         # JWT, RBAC, tenant isolation
├── tests/
│   ├── contract/             # Agent contract tests (REQUIRED)
│   ├── integration/          # Multi-service tests
│   └── unit/                 # Unit tests (recommended)
├── specs/                    # Feature specs (speckit output)
├── deploy/
│   ├── docker-compose/       # Development compose files
│   └── helm/                 # Kubernetes Helm charts
├── .specify/                 # Speckit config and templates
├── pyproject.toml
├── uv.lock
└── Dockerfile
```

### Async Patterns

All async code MUST follow these conventions:

```python
# CORRECT: async all the way down
async def dispatch_agent_task(
    request: AgentDispatchRequest,
    tenant: TenantContext = Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
) -> AgentDispatchResponse:
    async with TenantSession(db, tenant.id):
        config = await agent_registry.get(request.agent_type)
    task = await agent_task.kiq(request, tenant.id)
    return AgentDispatchResponse(task_id=task.task_id)

# FORBIDDEN: sync calls in async context
async def bad_handler():
    result = requests.get(url)  # BLOCKS event loop
    time.sleep(1)               # BLOCKS event loop
```

**Taskiq shared dependencies** (via `taskiq-fastapi`):

```python
from taskiq_fastapi import TaskiqDepends

@broker.task
async def agent_task(
    payload: dict, tenant_id: str
) -> dict:
    db: AsyncSession = TaskiqDepends(get_db)
    # Same DI container as FastAPI
```

### Dependency Injection Conventions

- All shared resources (DB session, cache, secrets client)
  provided via FastAPI `Depends()`
- No global mutable singletons outside app startup lifecycle
- Startup: `@app.on_event("startup")` initializes OTel, Loguru,
  broker; validates OpenBao connectivity
- Shutdown: `@app.on_event("shutdown")` flushes OTel spans,
  closes broker connection

### Error Handling

```python
# CORRECT: typed exceptions with structured context
class TenantNotFoundError(OpenFactoryError):
    error_code = "tenant_not_found"

class AgentDispatchError(OpenFactoryError):
    error_code = "agent_dispatch_failed"

# FastAPI exception handler translates to structured JSON:
# {
#   "error": {
#     "code": "tenant_not_found",
#     "message": "...",
#     "trace_id": "..."
#   }
# }

# FORBIDDEN:
try:
    ...
except Exception:  # too broad — defect
    pass           # swallowed error — defect
```

- All exceptions MUST carry `trace_id` (injected by middleware)
- 4xx errors: log at INFO level (expected client errors)
- 5xx errors: log at ERROR level with full stack trace
- Agent task failures: stored in Taskiq result with `error_code`
  and `trace_id`

### Code Quality

- Formatter: `ruff format` (enforced in CI)
- Linter: `ruff check` with rules: E, F, I, N, UP, ANN
  (type annotations required for public APIs)
- Type checking: `mypy` in strict mode for `src/openfactory/`
- Pre-commit hooks: ruff, mypy, no-secrets scanner
- Test runner: `pytest-asyncio` for all async tests

### Contribution Guidelines

1. All features start with a speckit spec (`/speckit.specify`)
2. Implementation plan required before coding (`/speckit.plan`)
3. Contract tests MUST be written and failing before P1
   implementation begins
4. PR requires: passing CI (lint + type check + contract tests),
   one peer review, constitution check completed (plan.md
   "Constitution Check" section signed off)
5. No feature merges with unresolved `[NEEDS CLARIFICATION]`
   in spec
6. Complexity justification required for any deviation from
   YAGNI principle (documented in plan.md)

## Open Questions / Deferred Decisions

### Architecture

1. **NATS JetStream Retention Policy**
   `TODO(NATS_RETENTION)`: Work-queue retention assumed (messages
   deleted after ack). Should compliance use cases require a
   separate replay stream? Decision deferred pending compliance
   requirements from first enterprise client.

2. **Graphiti Version + Agno Integration Path**
   `TODO(GRAPHITI_VERSION)`: Graphiti's Agno integration path is
   not formally documented. Confirm whether Graphiti is used as a
   direct client within CRM agent code or wrapped as an Agno tool.

3. **Multi-Region Strategy**: Single-region deployment assumed
   for v1. Multi-region active-active raises questions for Qdrant
   shard routing, OpenBao replication, and NATS geo-clustering.
   Deferred to post-v1.

4. **Ollama Deployment Model**
   `TODO(OLLAMA_DEPLOYMENT)`: Is Ollama deployed inside K8s
   (GPU node pool) or external? Resource requirements and GPU
   scheduling strategy not defined.

### Security

5. **MCP Server Auth Token Forwarding**
   `TODO(COPILOTKIT_AUTH)`: CopilotKit AG-UI frontend forwarding
   strategy for Zitadel OAuth2 tokens to MCP servers via token
   introspection needs a concrete implementation spec.

6. **Agent-to-Agent JWT Rotation**: Service account JWTs for
   agent-to-agent auth are stored in OpenBao. Rotation frequency
   and zero-downtime rotation procedure not yet specified.

### Operations

7. **Canary Rollout Policy**
   `TODO(CANARY_POLICY)`: Canary percentages, observation windows,
   and automated rollback triggers for OpenFactory service
   deployments not defined.

8. **Tenant Offboarding**
   `TODO(TENANT_OFFBOARDING)`: Data retention, deletion, and
   archival procedure for churned tenants not specified. GDPR
   implications for Mem0 and Qdrant data not addressed.

9. **Docling Audio/Image GPU Requirements**: Audio and image
   processing via Docling may require GPU resources. GPU
   scheduling strategy for ingestion workers not defined.

10. **LLM-as-Judge Eval Automation**: Langfuse LLM-as-judge
    evaluations are listed as a feature but eval dataset, scoring
    criteria, and automated trigger conditions are not specified.

## Governance

This constitution supersedes all other project practices, README
instructions, and verbal agreements regarding architecture and
engineering standards. No implementation decision may contradict
a NON-NEGOTIABLE principle without a formal amendment.

### Amendment Procedure

1. Open a GitHub issue with label `constitution-amendment`
   describing the proposed change
2. Provide: motivation, affected components, migration plan for
   existing code, constitution version bump type
   (MAJOR/MINOR/PATCH)
3. Require: minimum 2 maintainer approvals; no objections within
   5 business days
4. Update constitution file; update all affected templates;
   update `.specify/memory/` files
5. Merge with commit:
   `docs: amend constitution to vX.Y.Z (<summary>)`

### Versioning Policy

- **MAJOR**: Backward-incompatible removal or redefinition of
  any NON-NEGOTIABLE principle, or removal of a confirmed stack
  component
- **MINOR**: Addition of a new principle, new architecture
  section, or material expansion of existing guidance
- **PATCH**: Clarifications, wording improvements, typo fixes,
  example updates, addition of open questions

### Compliance Review

- All feature plans MUST include a "Constitution Check" section
  (see plan-template.md)
- All PRs MUST be reviewed against constitution compliance
  before merge
- Quarterly architecture review: assess open questions, retire
  resolved items, bump version if needed

**Version**: 2.0.0 | **Ratified**: 2026-03-04 | **Last Amended**: 2026-03-15
