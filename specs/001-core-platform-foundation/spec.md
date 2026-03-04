# Feature Specification: Core Platform Foundation

**Feature Branch**: `001-core-platform-foundation`
**Created**: 2026-03-04
**Status**: Draft
**Input**: User description: "Create specs to start implementing the project based on the constitution"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Platform Engineer Bootstraps the Project (Priority: P1)

A platform engineer clones the OpenFactory repository, runs the
development environment setup, and has a fully functional local
stack with a running API server, connected database, cache, task
queue, and health checks passing. They can verify the stack is
operational by calling the health endpoint and seeing all
dependencies report healthy.

**Why this priority**: Nothing else can be built until the project
scaffolding, dependency configuration, database connectivity, and
local development environment exist. This is the foundation for
every subsequent feature.

**Independent Test**: Can be fully tested by running the dev
environment and confirming `GET /v1/health` returns 200 with all
dependency statuses healthy. Delivers a runnable skeleton that
proves the stack choices work together.

**Acceptance Scenarios**:

1. **Given** a fresh clone of the repository, **When** the
   engineer runs the dev environment setup command, **Then** all
   services (API server, database, cache, message broker) start
   successfully within 3 minutes.
2. **Given** the local stack is running, **When** the engineer
   calls `GET /v1/health`, **Then** the response is 200 with
   status for each dependency (database, cache, message broker).
3. **Given** the local stack is running, **When** the engineer
   visits `/docs`, **Then** the auto-generated API documentation
   is accessible and lists all available endpoints.
4. **Given** the project is cloned, **When** the engineer runs
   the linter and type checker, **Then** all checks pass with
   zero errors.

---

### User Story 2 - Authenticated User Dispatches an Agent Task (Priority: P2)

An authenticated enterprise user sends a request to dispatch an
agent task. The system validates their identity, checks their
tenant permissions, places the task on the queue, and returns a
task ID. The user can then poll for task status or connect to the
SSE stream to watch progress. Even though no real agents exist
yet, the full request-response lifecycle works end-to-end with a
stub "echo" agent that returns the input payload.

**Why this priority**: Validates the complete request lifecycle:
authentication, tenant isolation, task dispatch via queue, result
retrieval, and SSE streaming. The echo agent proves the factory
pattern works before real agents are built.

**Independent Test**: Can be tested by authenticating, dispatching
a task to the echo agent, and verifying the task completes with
the echoed payload. Delivers a working end-to-end pipeline that
all future agents will use.

**Acceptance Scenarios**:

1. **Given** a valid authenticated user with the `agent_operator`
   role, **When** they submit `POST /v1/agents/dispatch` with a
   valid payload, **Then** they receive a 202 response with a
   `task_id` and a `Location` header pointing to the task
   resource, plus HATEOAS links for polling and streaming.
2. **Given** a dispatched task, **When** the user polls
   `GET /v1/agents/tasks/{task_id}`, **Then** they see the task
   progress through `PENDING` → `RUNNING` → `SUCCESS` states.
3. **Given** a dispatched task, **When** the user connects to
   `GET /v1/agents/tasks/{task_id}/stream`, **Then** they
   receive SSE events showing real-time task progress.
4. **Given** an unauthenticated request, **When** calling any
   protected endpoint, **Then** the system returns 401 with a
   structured error response including `trace_id`.
5. **Given** a user from tenant A, **When** they attempt to
   access a task belonging to tenant B, **Then** the system
   returns 404 (not 403, to avoid information leakage).

---

### User Story 3 - Platform Engineer Registers and Manages Agent Types (Priority: P3)

A platform engineer with `tenant_admin` role registers new agent
types in the system, views the registry of available agents,
updates agent configurations, and retires agents that are no
longer needed. The registry serves as the source of truth for
what agents the factory can dispatch.

**Why this priority**: The agent registry is required before real
specialized agents can be added. It establishes the data model
and API for agent lifecycle management and validates CRUD
operations with tenant isolation.

**Independent Test**: Can be tested by registering an agent type,
listing it, updating its configuration, and retiring it. Delivers
a working registry CRUD with pagination, HATEOAS links, and
tenant scoping.

**Acceptance Scenarios**:

1. **Given** an admin user, **When** they `POST /v1/agents/registry`
   with agent type details, **Then** the system creates the
   registry entry and returns 201 with a `Location` header and
   HATEOAS links.
2. **Given** registered agent types, **When** the admin calls
   `GET /v1/agents/registry`, **Then** they receive a paginated
   list of agents for their tenant only, with HATEOAS links on
   each entry.
3. **Given** a registered agent type, **When** the admin calls
   `DELETE /v1/agents/registry/{agent_type}`, **Then** the agent
   is marked inactive and returns 204. In-flight tasks for that
   agent type complete, but no new dispatches are accepted.
4. **Given** agents registered by tenant A, **When** tenant B
   lists the registry, **Then** tenant B sees only their own
   registered agents.

---

### User Story 4 - Observability Signals Emitted for Every Operation (Priority: P4)

Every API request, task dispatch, and background operation
produces structured logs, OTel spans, and Prometheus metrics
automatically. A platform engineer can correlate a user request
to its task execution by following the `trace_id` across logs
and traces.

**Why this priority**: Observability is a NON-NEGOTIABLE
constitution principle. Building it into the foundation prevents
the "bolt-on observability" anti-pattern where signals are added
inconsistently after the fact.

**Independent Test**: Can be tested by dispatching a task and
verifying that logs contain `trace_id`, `tenant_id`, `agent_id`;
that Prometheus metrics are exposed at `/metrics`; and that OTel
spans are emitted to the configured collector endpoint.

**Acceptance Scenarios**:

1. **Given** any API request, **When** it completes, **Then** a
   structured JSON log record is written to stdout containing
   `trace_id`, `span_id`, `tenant_id`, `method`, `route`,
   `status_code`, and `duration_ms`.
2. **Given** a running API server, **When** calling
   `GET /metrics`, **Then** Prometheus metrics are exposed
   including `openfactory_http_requests_total` and
   `openfactory_http_request_duration_seconds`.
3. **Given** a dispatched agent task, **When** it completes,
   **Then** the task worker emits an OTel span with attributes
   `agent_type`, `tenant_id`, `task_id`, and `status`.
4. **Given** a request with a `traceparent` header, **When** it
   triggers a background task, **Then** the task's logs and
   spans share the same `trace_id` for end-to-end correlation.

---

### Edge Cases

- What happens when the database is unavailable at startup?
  The application MUST fail fast and not start.
- What happens when the message broker is unavailable during
  dispatch? The dispatch endpoint returns 503 with a structured
  error and `trace_id`.
- What happens when a task exceeds the 10-minute timeout?
  The task status transitions to `TIMEOUT` and the result
  records the timeout reason.
- What happens when a user dispatches to a non-existent agent
  type? The system returns 422 with error code
  `agent_type_not_found`.
- What happens when bulk dispatch contains a mix of valid and
  invalid items? Each item is processed independently; the
  response reports per-item outcomes.
- What happens when Redis cache is unavailable? The system
  degrades gracefully — agent registry lookups fall back to
  database; task results are polled from the database.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide a runnable local development
  environment via a single setup command that starts all
  dependencies (database, cache, message broker).
- **FR-002**: System MUST expose a health endpoint at
  `GET /v1/health` that reports individual dependency statuses.
- **FR-003**: System MUST validate authentication tokens on all
  protected endpoints and reject unauthenticated requests with
  401 and a structured error body.
- **FR-004**: System MUST enforce tenant isolation on all
  data operations — a user from one tenant MUST NOT see, modify,
  or infer the existence of another tenant's data.
- **FR-005**: System MUST accept agent task dispatch requests at
  `POST /v1/agents/dispatch`, validate the payload, enqueue the
  task asynchronously, and return 202 with a task ID.
- **FR-006**: System MUST support task status polling at
  `GET /v1/agents/tasks/{task_id}` returning current status and
  result when complete.
- **FR-007**: System MUST support real-time task streaming at
  `GET /v1/agents/tasks/{task_id}/stream` via Server-Sent Events.
- **FR-008**: System MUST provide an agent registry CRUD API at
  `/v1/agents/registry` with list, create, read, update, and
  delete operations.
- **FR-009**: All list endpoints MUST return paginated responses.
- **FR-010**: All resource responses MUST include HATEOAS links
  (`_links` object) with at minimum a `self` link and relevant
  state-transition links.
- **FR-011**: System MUST support bulk dispatch at
  `POST /v1/agents/dispatch/bulk` with per-item success/failure
  reporting.
- **FR-012**: System MUST emit structured JSON logs to stdout
  for every request and task, including `trace_id`, `span_id`,
  `tenant_id`.
- **FR-013**: System MUST expose Prometheus metrics at `/metrics`
  for request counts, durations, and task counts.
- **FR-014**: System MUST emit OTel spans for every API request
  and task execution.
- **FR-015**: System MUST include a stub "echo" agent that
  returns the input payload, registered in the factory and
  usable for end-to-end testing.
- **FR-016**: System MUST return all error responses in a
  consistent structured format:
  `{"error": {"code": "...", "message": "...", "trace_id": "..."}}`.
- **FR-017**: System MUST use fast JSON serialization as the
  default response format for all endpoints.
- **FR-018**: System MUST propagate trace context across async
  task boundaries so that a request's `trace_id` flows from the
  API handler to the task worker.

### Key Entities

- **Tenant**: An enterprise client organization. Has an ID,
  name, status, and configuration. All data is scoped to a
  tenant.
- **User**: A human or service account authenticated via the
  identity provider. Belongs to a tenant. Has roles that
  determine permissions.
- **Agent Registry Entry**: A registered agent type (e.g.,
  "echo", "email", "crm"). Has capabilities, supported
  providers, memory configuration, and active status. Scoped
  to a tenant with possible per-tenant overrides.
- **Agent Task**: A dispatched unit of work. Has an ID, agent
  type, tenant, status (pending/running/success/failed/timeout),
  payload, result, timestamps, and the provider used.
- **Task Result**: The output of a completed agent task.
  Includes the response data, token counts, duration, and
  error details if failed.

### Assumptions

- Local development uses container-based services (database,
  cache, broker) started via a compose file. Production uses
  managed/operator-deployed services per the constitution.
- Authentication in local development uses a simplified token
  validation that mimics the production IAM behavior without
  requiring a full identity provider deployment.
- The echo agent is sufficient to validate the entire dispatch
  pipeline; real specialized agents are separate features.
- Bulk operation max batch size defaults to 100 for tasks and
  20 for registry operations.
- Task timeout is 10 minutes (600 seconds) per the constitution.
- Agent registry cache TTL is 5 minutes per the constitution.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new contributor can go from fresh clone to
  running local stack with passing health check in under
  5 minutes.
- **SC-002**: An authenticated user can dispatch a task and
  receive a completed result from the echo agent within
  5 seconds on the local development environment.
- **SC-003**: All API endpoints return responses that include
  HATEOAS `_links` enabling full API navigation from the root
  endpoint without prior knowledge of URL patterns.
- **SC-004**: 100% of API requests produce a structured log
  record with `trace_id`, and the same `trace_id` appears in
  the corresponding task worker logs.
- **SC-005**: Prometheus metrics are available for every
  endpoint, and request count increments are visible after
  each API call.
- **SC-006**: A user from tenant A cannot access, list, or
  infer the existence of tenant B's data across all endpoints
  (tasks, registry, collections).
- **SC-007**: Bulk dispatch of 100 tasks completes within
  10 seconds and reports per-item outcomes.
- **SC-008**: All linting, type checking, and contract tests
  pass in CI with zero errors.
