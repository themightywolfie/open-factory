# OpenFactory

> Multi-tenant B2B SaaS AI agent factory orchestrator.

This repository implements the OpenFactory runtime and conventions: an async-first, multi-tenant agent factory with strong tenant isolation, observability, and a locked technology stack.

**Core Principles (short)**
- **Async-first:** All API endpoints, tasks and agents use `async`/`await`. No blocking I/O in request or agent paths.
- **Zero Static Credentials:** No secrets or API keys in source. Use OpenBao for runtime secrets; `.env.local` only for local dev.
- **Tenant Isolation:** RLS in Postgres, tenant-prefixed Qdrant collections and Redis keys, tenant-scoped memory namespaces.
- **Observability:** Every API call and agent task must emit structured logs (Loguru JSON), OTel spans, and Prometheus metrics.
- **Agent Factory Pattern:** All agent invocation goes through the factory dispatcher; never call agents directly from API handlers.
- **Locked Stack:** The project requires specific components (FastAPI, SQLAlchemy async, Agno, Mem0, Qdrant, Taskiq + NATS, Loguru, OpenBao, Zitadel, Docling, Langfuse). Substitutions are not permitted without a formal constitution amendment.

Key design constraints and conventions are defined in the constitution document at `.specify/memory/constitution.md` and summarized in `CLAUDE.md`. Those documents are authoritative.

Project layout (high level)

 - `src/openfactory/` — application code (API, agents, services, memory, observability)
 - `tests/` — contract, integration, and unit tests
 - `deploy/` — docker-compose, helm charts

API conventions

 - Use `ORJSONResponse` as the default FastAPI response class.
 - All list endpoints must paginate using `fastapi-pagination`.
 - REST endpoints follow HATEOAS: resource responses MUST include a `_links` object with at least a `self` link.
 - Agent dispatch endpoints are asynchronous: `POST /v1/agents/dispatch` returns `202 Accepted` and a `Location` header.

Key endpoints

 - `POST /v1/agents/dispatch` — submit an agent task (202)
 - `POST /v1/agents/dispatch/bulk` — bulk dispatch (100 max)
 - `GET /v1/agents/tasks` — list tasks (paginated)
 - `GET /v1/agents/tasks/{task_id}` — task status
 - `GET /v1/health` — liveness/readiness

Testing and workflow

 - Contract tests for agent I/O schemas must be written and failing before implementing P1 features.
 - Integration tests are required for P1 user stories. Do not assert on LLM content — assert on contract shape and side effects.
 - Format with `ruff format`; lint with `ruff check`; types with `mypy --strict` on `src/openfactory/`.

Local development quickstart

1. Create a local `.env.local` for any non-sensitive dev overrides (git-ignored).
2. Follow `deploy/docker-compose` for local infra (Postgres, Redis, Qdrant, NATS). The repo's compose manifests are the recommended dev setup.
3. Run the FastAPI app (example):

```powershell
# from repository root
uv run -m src.openfactory.main
```

Contribution

 - Follow the Constitution workflow: start with `/speckit.specify`, add a plan with `/speckit.plan`, write failing contract tests, implement, then open a PR with CI passing (lint, types, tests) and a constitution check.
 - All PRs must preserve tenant isolation and observability instrumentation.

Where to read more

 - Constitution: [.specify/memory/constitution.md](.specify/memory/constitution.md)
 - Claude summary: [CLAUDE.md](CLAUDE.md)

License & Code of Conduct

 - See repository metadata for license and contributor guidelines.
