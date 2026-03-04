# OpenFactory — Claude Code Rules

> Derived from `.specify/memory/constitution.md` v1.0.0.
> Constitution is the source of truth. If conflict, constitution wins.

## Project

Multi-tenant B2B SaaS AI agent factory orchestrator. Python backend,
React/Next.js frontend. All architecture decisions are in the constitution.

## NON-NEGOTIABLE Rules

These are blocking defects. Never violate them.

### 1. Async-First

- All API endpoints, task handlers, and agent code MUST use `async`/`await`
- NEVER use `requests`, `time.sleep()`, synchronous DB calls, or blocking
  I/O in async context
- Use `httpx` (async) instead of `requests`
- Use `asyncio.sleep()` instead of `time.sleep()`

### 2. Zero Static Credentials

- NEVER put secrets, API keys, passwords, or tokens in source code
- NEVER create `.env` files (only `.env.local` which is git-ignored)
- All secrets come from OpenBao at runtime
- If writing example config, use `<PLACEHOLDER>` not real-looking values

### 3. Tenant Isolation

- PostgreSQL: all tenant tables use RLS keyed on `tenant_id`
- Always use `TenantSession` context manager, NEVER raw `Session`
- Qdrant collections: `{tenant_id}_{agent_type}_{collection_name}`
- Redis keys: `{tenant_id}:` prefix via `TenantCache`, NEVER raw `redis.get()`
- Mem0 namespace: `{tenant_id}/{user_id}/{agent_id}`
- NATS subjects: `tasks.{tenant_id}.{agent_type}`

### 4. Observability

- Every API endpoint, agent task, and background task MUST emit:
  - Loguru JSON log with `trace_id`, `span_id`, `tenant_id`, `agent_id`
  - OTel span (LLM → Langfuse, infra → Tempo)
  - Prometheus metric (count + duration histogram)
- NEVER ship a feature without observability instrumentation

### 5. Agent Factory Pattern

- ALL agent invocation goes through the factory dispatcher
- NEVER call agents directly from API handlers
- Agent types registered in `agent_registry` table (cached in Redis)

### 6. Locked Stack

Do NOT substitute any of these. No alternatives permitted:

| Concern | Technology |
|---------|-----------|
| API | FastAPI (async) + ORJSONResponse (default) + fastapi-pagination |
| ORM | SQLAlchemy (async) |
| Validation | Pydantic |
| Agents | Agno |
| Memory | Mem0 (semantic), Graphiti (CRM temporal) |
| Vector DB | Qdrant |
| Task queue | Taskiq + NATS JetStream |
| Logging | Loguru |
| Secrets | OpenBao |
| IAM | Zitadel |
| Doc parsing | Docling + HybridChunker |
| LLM tracing | Langfuse |
| Infra observability | Grafana LGTM stack |
| UI | CopilotKit + AG-UI (React/Next.js) |

## REST API Design — Richardson Maturity Model Level 3 (HATEOAS)

All REST endpoints MUST conform to Richardson Maturity Model Level 3:

1. **Level 1 — Resources**: Every entity has a distinct URI
2. **Level 2 — HTTP Verbs**: Correct use of GET, POST, PUT, PATCH, DELETE
3. **Level 3 — HATEOAS**: Responses include hypermedia links for
   discoverability and state transitions

### HATEOAS Response Pattern

Every resource response MUST include a `_links` object with at minimum
a `self` link. Include relevant state-transition and related-resource
links.

```python
class HATEOASLink(BaseModel):
    href: str
    method: str = "GET"
    title: str | None = None

class TaskResponse(BaseModel):
    task_id: str
    status: TaskStatus
    agent_type: str
    tenant_id: str
    # ... other fields

    _links: dict[str, HATEOASLink]

    @computed_field
    @property
    def _links(self) -> dict[str, HATEOASLink]:
        links = {
            "self": HATEOASLink(
                href=f"/v1/agents/tasks/{self.task_id}",
            ),
            "stream": HATEOASLink(
                href=f"/v1/agents/tasks/{self.task_id}/stream",
                title="SSE event stream",
            ),
            "cancel": HATEOASLink(
                href=f"/v1/agents/tasks/{self.task_id}/cancel",
                method="POST",
                title="Cancel task",
            ),
        }
        if self.status == "success":
            links["result"] = HATEOASLink(
                href=f"/v1/agents/tasks/{self.task_id}/result",
                title="Task result",
            )
        return links
```

### HTTP Verb Usage

| Verb | Purpose | Idempotent | Response Code |
|------|---------|------------|---------------|
| GET | Read resource(s) | Yes | 200 |
| POST | Create resource / trigger action | No | 201 (create) or 202 (async) |
| PUT | Full resource replacement | Yes | 200 |
| PATCH | Partial update | No | 200 |
| DELETE | Remove resource | Yes | 204 |

- POST for agent dispatch returns **202 Accepted** (async task)
- All create operations return a `Location` header with the new
  resource URI
- DELETE returns 204 with no body

### Bulk Operations

Endpoints that operate on collections MUST support bulk operations
where it makes sense. Bulk endpoints reduce round-trips for
enterprise clients managing many agents/documents/tasks.

**Bulk endpoint convention**: `POST /v1/{resource}/bulk/{action}`

```python
# --- Bulk dispatch multiple agent tasks ---
class BulkDispatchRequest(BaseModel):
    items: list[AgentDispatchRequest] = Field(..., max_length=100)

class BulkDispatchResponseItem(BaseModel):
    index: int
    task_id: str | None = None
    status: Literal["accepted", "failed"]
    error: ErrorDetail | None = None
    _links: dict[str, HATEOASLink] = {}

class BulkDispatchResponse(BaseModel):
    total: int
    accepted: int
    failed: int
    items: list[BulkDispatchResponseItem]

@router.post(
    "/v1/agents/dispatch/bulk",
    response_model=BulkDispatchResponse,
    status_code=202,
)
async def bulk_dispatch(request: BulkDispatchRequest, ...):
    ...

# --- Bulk cancel tasks ---
class BulkCancelRequest(BaseModel):
    task_ids: list[str] = Field(..., max_length=100)

@router.post("/v1/agents/tasks/bulk/cancel", ...)
async def bulk_cancel(request: BulkCancelRequest, ...):
    ...

# --- Bulk ingest documents ---
class BulkIngestRequest(BaseModel):
    items: list[IngestRequest] = Field(..., max_length=50)

@router.post("/v1/knowledge/ingest/bulk", status_code=202, ...)
async def bulk_ingest(request: BulkIngestRequest, ...):
    ...

# --- Bulk delete knowledge collections ---
@router.post("/v1/knowledge/collections/bulk/delete", ...)
async def bulk_delete_collections(
    request: BulkDeleteRequest, ...
):
    ...
```

**Bulk operation rules**:

- Max batch size enforced per endpoint (100 tasks, 50 documents)
- Each item processed independently — partial success is valid
- Response always reports per-item outcome (`accepted`/`failed`)
- Failed items include `error` with `error_code` and `trace_id`
- Bulk operations are dispatched as a single Taskiq task that
  fans out internally (not N separate task dispatches)

### Endpoints That MUST Support Bulk

| Endpoint | Bulk Variant | Max Batch |
|----------|-------------|-----------|
| `POST /v1/agents/dispatch` | `POST /v1/agents/dispatch/bulk` | 100 |
| `POST /v1/agents/tasks/{id}/cancel` | `POST /v1/agents/tasks/bulk/cancel` | 100 |
| `POST /v1/knowledge/ingest` | `POST /v1/knowledge/ingest/bulk` | 50 |
| `DELETE /v1/knowledge/collections/{id}` | `POST /v1/knowledge/collections/bulk/delete` | 50 |
| `POST /v1/agents/registry` | `POST /v1/agents/registry/bulk` | 20 |

## FastAPI Conventions

### Default Response Class

Use `ORJSONResponse` as the default response class for all FastAPI apps.
Faster JSON serialization than stdlib `json`.

```python
from fastapi import FastAPI
from fastapi.responses import ORJSONResponse

app = FastAPI(default_response_class=ORJSONResponse)
```

### Pagination

Use `fastapi-pagination` for all list endpoints. Always paginate —
never return unbounded result sets.

```python
from fastapi import APIRouter
from fastapi_pagination import Page, add_pagination
from fastapi_pagination.ext.sqlalchemy import paginate

router = APIRouter()

@router.get("/v1/agents/tasks", response_model=Page[TaskResponse])
async def list_tasks(
    tenant: TenantContext = Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
) -> Page[TaskResponse]:
    async with TenantSession(db, tenant.id):
        return await paginate(db, select(Task))

# Call add_pagination(app) after including all routers
add_pagination(app)
```

## Code Patterns

### Python

```python
# Package manager: uv (NEVER pip/poetry/conda)
# Python 3.12+ required

# Async handler pattern
async def my_endpoint(
    request: MyRequest,
    tenant: TenantContext = Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
) -> MyResponse:
    async with TenantSession(db, tenant.id):
        ...

# Taskiq task pattern
@broker.task
async def my_task(payload: dict, tenant_id: str) -> dict:
    db: AsyncSession = TaskiqDepends(get_db)
    ...

# Error handling — typed exceptions only
class MyError(OpenFactoryError):
    error_code = "my_error"

# FORBIDDEN patterns:
# - bare `except Exception: pass`
# - `except Exception` without re-raise or specific handling
# - global mutable singletons outside app startup
```

### Error Responses

```json
{"error": {"code": "tenant_not_found", "message": "...", "trace_id": "..."}}
```

- 4xx → log at INFO
- 5xx → log at ERROR with full stack trace

### Code Quality

- Format: `ruff format`
- Lint: `ruff check` (rules: E, F, I, N, UP, ANN)
- Types: `mypy --strict` on `src/openfactory/`
- Tests: `pytest-asyncio`
- Pre-commit: ruff, mypy, no-secrets scanner

## Project Structure

```
src/openfactory/
├── api/              # FastAPI routers, deps, middleware
├── agents/           # Agent definitions
│   ├── factory/      # Dispatcher agent
│   ├── email/        # Email agent
│   ├── crm/          # CRM agent (uses Graphiti)
│   ├── cdp/          # CDP agent
│   ├── coding/       # Coding agent
│   ├── research/     # Research/RAG agent
│   └── productivity/ # Productivity agent
├── knowledge/        # Docling ingestion pipeline
├── memory/           # Mem0 + Graphiti wrappers
├── models/           # SQLAlchemy models
├── schemas/          # Pydantic request/response schemas
├── services/         # Business logic layer
├── tasks/            # Taskiq task definitions
├── config/           # Configuration (OpenBao-backed)
├── observability/    # OTel, Loguru, metrics setup
└── security/         # JWT, RBAC, tenant isolation

tests/
├── contract/         # Agent contract tests (REQUIRED before P1 impl)
├── integration/      # Multi-service integration tests
└── unit/             # Unit tests (recommended)

deploy/
├── docker-compose/   # Dev compose files
└── helm/             # K8s Helm charts
```

## LLM Providers

All five MUST be supported. Never hardcode a single provider.

- Azure OpenAI
- OpenAI
- Anthropic
- Google Gemini
- Ollama (local/self-hosted)

Provider selection is per-request and per-agent-type with fallback.

## Testing Rules

- Contract tests (schemas, tool signatures) MUST exist and FAIL
  before implementing P1 features
- Integration tests REQUIRED for P1 user stories
- NEVER assert on LLM response content — assert on contract shape,
  status codes, and side effects
- Red-Green-Refactor enforced for contract + integration tests

## Workflow

1. Feature starts with `/speckit.specify`
2. Plan with `/speckit.plan`
3. Contract tests written and failing
4. Implement
5. PR requires: CI pass (lint + types + tests), peer review,
   constitution check

## Key API Endpoints

All responses include HATEOAS `_links` for discoverability.

**Agents**:
- `POST /v1/agents/dispatch` — Submit agent task (202)
- `POST /v1/agents/dispatch/bulk` — Bulk dispatch (202)
- `GET /v1/agents/tasks` — List tasks (paginated)
- `GET /v1/agents/tasks/{task_id}` — Get task status
- `GET /v1/agents/tasks/{task_id}/stream` — SSE stream (AG-UI)
- `GET /v1/agents/tasks/{task_id}/result` — Get task result
- `POST /v1/agents/tasks/{task_id}/cancel` — Cancel task
- `POST /v1/agents/tasks/bulk/cancel` — Bulk cancel

**Agent Registry**:
- `GET /v1/agents/registry` — List agent types (paginated)
- `POST /v1/agents/registry` — Register agent type (201)
- `GET /v1/agents/registry/{agent_type}` — Get agent type
- `PUT /v1/agents/registry/{agent_type}` — Update agent type
- `DELETE /v1/agents/registry/{agent_type}` — Retire agent (204)

**Knowledge**:
- `POST /v1/knowledge/ingest` — Ingest document (202)
- `POST /v1/knowledge/ingest/bulk` — Bulk ingest (202)
- `GET /v1/knowledge/collections` — List collections (paginated)
- `GET /v1/knowledge/collections/{id}` — Get collection
- `DELETE /v1/knowledge/collections/{id}` — Delete collection (204)
- `POST /v1/knowledge/collections/bulk/delete` — Bulk delete

**System**:
- `GET /v1/health` — Liveness/readiness
- `GET /v1/` — API root with HATEOAS links to all top-level resources

## What NOT To Do

- Never bypass factory dispatcher for agent calls
- Never use raw SQL without tenant RLS context
- Never create `.env` files in VCS
- Never add sync blocking calls in async code
- Never substitute locked stack components
- Never ship features without OTel + metrics + logs
- Never swallow exceptions (`except: pass`)
- Never use `requests` library (use `httpx`)
- Never access Redis/Qdrant/Mem0 without tenant prefix/namespace
- Never hardcode LLM provider assumptions

## Reference Documents

- **Constitution**: `.specify/memory/constitution.md` (authoritative)
- **Figma rules**: `.claude/figma-design-system-rules.md`
- **Speckit templates**: `.specify/templates/`
