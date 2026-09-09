# FlowForge

FlowForge is a workflow automation platform designed around a custom workflow execution engine.

The project is intentionally developed version by version. Each version solves a concrete limitation of the previous version while remaining a working product.

## Current Version

**V0.6 — Asynchronous Execution**

V0.1 established the backend foundation and basic product surface. V0.2 adds the first persistent visual workflow-definition system.

## V0.1 — Foundation

V0.1 is complete and verified.

Implemented:

- Java 25 + Spring Boot 4.1.1 backend
- PostgreSQL persistence through Docker Compose
- Flyway database migrations
- User creation and retrieval
- User validation and duplicate-email handling
- Workflow creation, retrieval, listing, update, and deletion
- Standardized API error responses
- Health endpoint
- React 19 + TypeScript + Vite frontend
- Development-user creation/reuse through browser local storage
- Dashboard with workflow count
- Workflow listing, creation, and deletion
- Backend/frontend browser integration
- Backend integration tests
- Frontend production build
- Frontend linting

## V0.2 — Visual Workflow Builder

### Goal

V0.2 answers:

> Can FlowForge represent and persist a workflow as a visual graph?

### Implemented and manually verified

- React Flow / XYFlow workflow canvas
- Persistent Start node for newly created workflows
- Migration of existing workflow versions to include a Start node
- HTTP Request node
- Transform node
- Node movement
- Node creation
- Node deletion
- Start-node deletion protection
- Edge creation
- Individual edge selection and deletion
- Automatic removal of incident edges when a node is deleted
- HTTP Request configuration:
  - method
  - URL
- Transform configuration:
  - expression
- Save workflow definition
- Load workflow definition
- PostgreSQL persistence of workflow versions, nodes, and edges
- Preservation of node IDs and edge IDs during save/load
- Replacement of the current graph on save
- Backend integration tests for workflow-definition behavior
- Frontend production build
- Frontend linting

### Current V0.2 graph model

```text
Workflow
   |
   +-- WorkflowVersion
          |
          +-- WorkflowNode
          |      +-- Start
          |      +-- HTTP Request
          |      +-- Transform
          |
          +-- WorkflowEdge
```

The current workflow-definition API is:

```text
GET /api/workflows/{workflowId}/definition
PUT /api/workflows/{workflowId}/definition
```

See [`docs/api/workflow-definition.md`](docs/api/workflow-definition.md).

## V0.3 — Workflow Validation

### Goal

V0.3 answers:

> Can FlowForge reject structurally invalid workflow definitions before they replace a persisted valid graph?

The answer is yes.

Validation is implemented as a separate, reusable backend concern. The workflow-definition service validates the submitted graph before deleting or replacing the existing persisted graph.

### Implemented and verified

- Dedicated `WorkflowDefinitionValidator`
- Dedicated `ValidationResult` and `ValidationError` response models
- Dedicated `WorkflowDefinitionValidationException`
- Exactly one Start node required
- Supported node types restricted to:
  - `start`
  - `httpRequest`
  - `transform`
- Duplicate node-ID detection
- Edge source/target existence validation
- Self-loop rejection
- Duplicate edge rejection
- HTTP Request configuration validation:
  - method required
  - method restricted to `GET`, `POST`, `PUT`, `DELETE`
  - URL required
  - URL must be a valid `http`/`https` URI with a host
- Transform expression validation
- Start-node configuration validation
- Disconnected-node detection through reachability from Start
- Multiple validation errors collected in one response
- Validation occurs before destructive persistence
- Invalid saves return HTTP 400 with structured validation details
- Existing valid definitions remain intact after rejected saves
- Validator unit-test coverage
- Workflow-definition controller integration-test coverage
- Full backend test suite passes
- Manual API verification completed

### Validation response

An invalid definition returns:

```json
{
  "valid": false,
  "errors": [
    {
      "nodeId": null,
      "message": "Workflow must contain a Start node."
    }
  ]
}
```

`nodeId` is `null` when an error applies to the workflow as a whole rather than to one specific node.

### Validation boundary

V0.3 validates workflow definitions but does not execute them.

It deliberately does not include:

- synchronous execution
- execution persistence
- asynchronous workers
- RabbitMQ
- retries
- idempotency
- scheduling
- branching
- variables or expression evaluation

Those capabilities remain later-version concerns.

## V0.4 — Synchronous Execution Engine

### Goal

V0.4 answers:

> Can FlowForge execute an already-validated workflow definition synchronously within the backend?

The answer is yes.

V0.4 introduces the first workflow execution path. The execution engine consumes the validated `WorkflowDefinition` produced by the workflow-definition layer and executes nodes in graph order during the API request.

### Implemented and verified

- `ExecutionContext` for trigger data, variables, node outputs, and metadata
- `NodeExecutionContext` for node-level execution
- `NodeExecutionResult` for explicit success/failure results
- Pluggable `WorkflowNodeExecutor` interface
- `WorkflowNodeExecutorRegistry`
- Start-node executor
- Transform executor
- HTTP Request executor
- Synchronous `WorkflowExecutionEngine`
- `WorkflowExecutionService` application layer
- Workflow execution REST endpoint
- Successful execution response mapping
- Runtime cycle detection
- Multiple-outgoing-edge rejection in V0.4
- Missing-executor handling
- Node execution failure handling
- Actual local HTTP execution tests
- Workflow execution controller integration tests
- Full backend test suite verification: **100 tests, 0 failures, 0 errors**

### Execution model

```text
POST /api/workflows/{workflowId}/execute
              ↓
WorkflowExecutionService
              ↓
WorkflowDefinitionService
              ↓
Validated WorkflowDefinition
              ↓
WorkflowExecutionEngine
              ↓
WorkflowNodeExecutorRegistry
       ┌──────┼──────────┐
       ↓      ↓          ↓
     Start Transform  HTTP Request
       └──────┼──────────┘
              ↓
       ExecutionContext
              ↓
   ExecuteWorkflowResponse
```

The engine follows edges from the Start node. A node's successful output is stored in `ExecutionContext.nodeOutputs`.

V0.4 supports a single linear execution path. If a node has more than one outgoing edge, execution fails explicitly because branching is a later concern.

### Supported execution behavior

`start` returns the workflow trigger data as its output.

`transform` currently returns its configured expression string. V0.4 does **not** introduce an expression language or arbitrary code execution; expression evaluation remains a future concern.

`httpRequest` performs synchronous HTTP requests using Java's built-in `HttpClient`. The current node configuration supports the methods and URL already defined by V0.3.

HTTP responses below status 400 are successful node results. HTTP 400 and above are execution failures.

### Important V0.4 boundary

V0.4 intentionally did not include execution persistence, execution history, asynchronous execution, worker processes, retries, idempotency, scheduling, branching, durable execution state, or expression evaluation.

V0.5 keeps the same synchronous execution engine and adds durable execution state and history around it.

## V0.5 — Execution Persistence

### Goal

V0.5 answers:

> Can FlowForge keep a durable record of workflow executions and their node-level outcomes while retaining the simple synchronous execution model?

The answer is yes.

### Implemented and verified

- PostgreSQL-backed `executions` table
- Workflow-version reference on every execution
- Execution lifecycle status: `RUNNING`, `SUCCESS`, `FAILED`
- Persisted trigger data
- Execution start/completion timestamps
- Persisted execution-level error message
- PostgreSQL-backed `execution_nodes` table
- Node lifecycle status: `RUNNING`, `SUCCESS`, `FAILED`
- Persisted node outputs and node-level errors
- PostgreSQL-backed `execution_events` table
- Execution lifecycle events
- Node start/completion/failure events
- Execution history API
- Execution detail API
- Stable execution ID returned by the execution endpoint
- Lifecycle listener boundary between the execution engine and persistence
- Synchronous execution remains unchanged in its request/response model
- Failure-path persistence integration coverage
- Full backend test suite verification: **100 tests, 0 failures, 0 errors**

### Persistence model

```text
WorkflowVersion
      |
      +---- Execution
               |
               +---- ExecutionNode
               |
               +---- ExecutionEvent
```

An execution stores the exact workflow-version ID used by the execution service. Node records and events reference execution IDs and node UUIDs.

### Execution lifecycle

```text
create execution
      |
      v
RUNNING
      |
      +--> NODE_STARTED
      |        |
      |        v
      |   node execution
      |        |
      |   +----+----+
      |   |         |
      | success   failure
      |   |         |
      |   v         v
      | NODE_     NODE_
      | COMPLETED FAILED
      |   |
      +---+
          |
          v
 EXECUTION_COMPLETED
          |
          v
       SUCCESS
```

A workflow failure records the failed execution and node state before the request returns.

### Execution history API

```text
GET /api/workflows/{workflowId}/executions
GET /api/executions/{executionId}
```

The workflow-scoped endpoint returns executions ordered newest first.

The detail endpoint returns the execution lifecycle state, trigger data, node execution records, and ordered execution events.

### Important V0.5 boundary

V0.5 remains synchronous.

It deliberately does not add:

- RabbitMQ
- asynchronous execution
- worker processes
- retries
- idempotency
- scheduling
- WebSockets
- Redis
- distributed execution

V0.5 also does not yet make workflow definitions immutable historical snapshots. Executions persist the workflow-version identity, while the current workflow-definition editing model remains mutable. Strong immutable versioning is a later architectural concern.

## Architecture

```text
React + TypeScript + Vite
          |
          | HTTP / REST
          v
     Spring Boot
          |
          v
      PostgreSQL
```

The backend remains a **modular monolith**. V0.5 keeps execution as a separate logical backend package and adds a persistence subpackage plus query API within the same Spring Boot application rather than introducing a separate worker service.

The architecture will become more distributed only when a concrete scaling, reliability, or execution requirement justifies it.

## Technology Stack

| Layer | Technology |
|---|---|
| Backend language | Java 25 |
| Backend framework | Spring Boot 4.1.1 |
| Build | Maven Wrapper |
| Persistence | Spring Data JPA / Hibernate |
| Database | PostgreSQL 17 |
| Migrations | Flyway 12 |
| Frontend | React 19 + TypeScript |
| Frontend build | Vite 8 |
| Workflow editor | React Flow / XYFlow |
| Local infrastructure | Docker Compose |

## Backend API

### Users

```text
POST /api/users
GET  /api/users/{id}
```

### Workflows

```text
POST   /api/users/{userId}/workflows
GET    /api/workflows/{id}
GET    /api/users/{userId}/workflows
PUT    /api/workflows/{id}
DELETE /api/workflows/{id}
```

### Workflow Definition

```text
GET /api/workflows/{workflowId}/definition
PUT /api/workflows/{workflowId}/definition
```

The definition endpoint persists the current visual graph, including node IDs, types, positions, configuration, and edge relationships.

### Workflow Execution

```text
POST /api/workflows/{workflowId}/execute
```

The endpoint creates a persisted execution and queues it for asynchronous processing. It returns the execution ID immediately with HTTP `202 Accepted`.

Optional trigger data can be supplied:

```json
{
  "triggerData": {
    "message": "hello"
  }
}
```

The request body may also be omitted; trigger data defaults to an empty map.

The response contains an `executionId` and its current status. Node outputs and terminal execution state are available through the execution-detail API.

Execution completion is persisted independently of the initial request. Clients can query the execution detail endpoint after the `202 Accepted` response.

Unknown workflow IDs remain HTTP 404 resource-not-found responses.

### Execution History

```text
GET /api/workflows/{workflowId}/executions
GET /api/executions/{executionId}
```

The first endpoint lists persisted executions for a workflow, newest first.

The second endpoint returns one execution with its trigger data, node execution records, and lifecycle events.

### Health

```text
GET /api/health
```

## Database

PostgreSQL is the system of record.

V0.1 introduced:

```text
users
workflows
```

V0.2 introduces:

```text
workflow_versions
workflow_nodes
workflow_edges
```

V0.4 does not introduce a database migration. Execution state remains in memory until V0.5 introduces execution persistence.\n\nThe migrations are:

```text
V1__create_users_and_workflows.sql
V2__create_workflow_definitions.sql
V3__add_start_nodes_to_workflows.sql
V4__create_execution_persistence.sql
```

**Applied migrations must not be edited.** Future schema changes require new migrations.

Workflow definitions are represented as a versioned graph. Each workflow currently has a current draft version (`version_number = 1`), with nodes and edges belonging to that version.

## Local Development

### Prerequisites

- JDK 25
- Docker Desktop
- Node.js / npm
- PostgreSQL provided through Docker Compose

### Backend

From `backend/`:

```powershell
.\mvnw.cmd clean test
```

Run the backend:

```powershell
.\mvnw.cmd spring-boot:run
```

Backend:

```text
http://localhost:8080
```

### V0.5 Execution Verification

The V0.4 execution path was verified with:

```powershell
.\mvnw.cmd test
```

The complete backend suite passed:

```text
Tests run: 100
Failures: 0
Errors: 0
Skipped: 0
BUILD SUCCESS
```

Additional V0.4 coverage includes:

- execution-engine unit tests
- executor-registry tests
- Start executor tests
- Transform executor tests
- HTTP executor tests against a controlled local HTTP server
- execution-service tests
- execution-response tests
- workflow execution controller integration tests
- persisted-definition execution through the REST endpoint

### Frontend

From `frontend/`:

```powershell
npm install
npm run dev
```

Frontend:

```text
http://localhost:5173
```

## Testing and Verification

### Backend

V0.5 backend verification completed successfully with:

```powershell
.\mvnw.cmd -q test
```

The full backend suite passed with:

```text
Failures: 0
Errors: 0
BUILD SUCCESS
```

Additional V0.3-focused verification includes:

- dedicated `WorkflowDefinitionValidator` unit tests
- workflow-definition controller integration tests
- valid workflow-definition save
- persisted definition retrieval
- invalid definition rejection with HTTP 400
- structured validation-error response
- preservation of the previously valid definition after an invalid save

The V0.2 workflow-definition coverage remains in the suite for:

- definition retrieval
- graph save/load
- node and edge persistence
- graph replacement
- edge-ID preservation
- invalid edge references
- unknown workflows

### Frontend

The latest frontend verification completed successfully:

```powershell
npm run build
npm run lint
```

Both commands are green.

### Manual V0.2 verification

The visual editor has been manually exercised for:

- creating HTTP Request and Transform nodes
- moving nodes
- creating edges
- configuring HTTP Request
- configuring Transform
- saving and refreshing
- persistence of node configuration
- deleting nodes
- automatic removal of connected edges
- Start-node protection
- selecting and deleting individual edges
- persistence of edge deletion

### Manual V0.3 verification

The workflow-definition API was manually verified against the running backend:

1. A valid `start → httpRequest` graph was saved successfully.
2. A subsequent GET returned the persisted graph.
3. An invalid graph containing no Start node returned HTTP 400.
4. The response contained:
   ```json
   {
     "valid": false,
     "errors": [
       {
         "nodeId": null,
         "message": "Workflow must contain a Start node."
       }
     ]
   }
   ```
5. A subsequent GET confirmed that the original valid graph remained persisted after the rejected save.

This verifies the validation-before-persistence safety boundary.

## Documentation

### V0.5 — Execution Persistence

### Goal

V0.5 answers:

> Can FlowForge keep a durable record of workflow executions and their node-level outcomes while retaining the simple synchronous execution model?

The answer is yes.

### Implemented and verified

- PostgreSQL-backed `executions` table
- Workflow-version reference on every execution
- Execution lifecycle status: `RUNNING`, `SUCCESS`, `FAILED`
- Persisted trigger data
- Execution start/completion timestamps
- Persisted execution-level error message
- PostgreSQL-backed `execution_nodes` table
- Node lifecycle status: `RUNNING`, `SUCCESS`, `FAILED`
- Persisted node outputs and node-level errors
- PostgreSQL-backed `execution_events` table
- Execution lifecycle events
- Node start/completion/failure events
- Execution history API
- Execution detail API
- Stable execution ID returned by the execution endpoint
- Lifecycle listener boundary between the execution engine and persistence
- Synchronous execution remains unchanged in its request/response model
- Failure-path persistence integration coverage
- Full backend test suite verification: **100 tests, 0 failures, 0 errors**

### Persistence model

```text
WorkflowVersion
      |
      +---- Execution
               |
               +---- ExecutionNode
               |
               +---- ExecutionEvent
```

An execution stores the exact workflow-version ID used by the execution service. Node records and events reference execution IDs and node UUIDs.

### Execution lifecycle

```text
create execution
      |
      v
RUNNING
      |
      +--> NODE_STARTED
      |        |
      |        v
      |   node execution
      |        |
      |   +----+----+
      |   |         |
      | success   failure
      |   |         |
      |   v         v
      | NODE_     NODE_
      | COMPLETED FAILED
      |   |
      +---+
          |
          v
 EXECUTION_COMPLETED
          |
          v
       SUCCESS
```

A workflow failure records the failed execution and node state before the request returns.

### Execution history API

```text
GET /api/workflows/{workflowId}/executions
GET /api/executions/{executionId}
```

The workflow-scoped endpoint returns executions ordered newest first.

The detail endpoint returns the execution lifecycle state, trigger data, node execution records, and ordered execution events.

### Important V0.5 boundary

V0.5 remains synchronous.

It deliberately does not add:

- RabbitMQ
- asynchronous execution
- worker processes
- retries
- idempotency
- scheduling
- WebSockets
- Redis
- distributed execution

V0.5 also does not yet make workflow definitions immutable historical snapshots. Executions persist the workflow-version identity, while the current workflow-definition editing model remains mutable. Strong immutable versioning is a later architectural concern.

## Architecture

- [`docs/architecture/v0.1-architecture.md`](docs/architecture/v0.1-architecture.md)
- [`docs/architecture/v0.2-architecture.md`](docs/architecture/v0.2-architecture.md)
- [`docs/architecture/v0.3-architecture.md`](docs/architecture/v0.3-architecture.md)
- [`docs/architecture/v0.4-architecture.md`](docs/architecture/v0.4-architecture.md)
- [`docs/architecture/v0.5-architecture.md`](docs/architecture/v0.5-architecture.md)

### API

- [`docs/api/workflow-definition.md`](docs/api/workflow-definition.md)
- [`docs/api/workflow-execution.md`](docs/api/workflow-execution.md)
- [`docs/api/execution-history.md`](docs/api/execution-history.md)

### Architecture Decision Records

- [`docs/decisions/ADR-001-java-spring-boot.md`](docs/decisions/ADR-001-java-spring-boot.md)
- [`docs/decisions/ADR-002-postgresql.md`](docs/decisions/ADR-002-postgresql.md)
- [`docs/decisions/ADR-003-modular-monolith.md`](docs/decisions/ADR-003-modular-monolith.md)
- [`docs/decisions/ADR-004-monorepo.md`](docs/decisions/ADR-004-monorepo.md)
- [`docs/decisions/ADR-005-flyway.md`](docs/decisions/ADR-005-flyway.md)
- [`docs/decisions/ADR-006-workflow-definition-validation.md`](docs/decisions/ADR-006-workflow-definition-validation.md)
- [`docs/decisions/ADR-007-synchronous-workflow-execution.md`](docs/decisions/ADR-007-synchronous-workflow-execution.md)
- [`docs/decisions/ADR-008-execution-persistence.md`](docs/decisions/ADR-008-execution-persistence.md)

### Diagrams

- [`docs/diagrams/workflow-definition-v0.2.md`](docs/diagrams/workflow-definition-v0.2.md)
- [`docs/diagrams/workflow-definition-v0.3-validation.md`](docs/diagrams/workflow-definition-v0.3-validation.md)
- [`docs/diagrams/workflow-execution-v0.4.md`](docs/diagrams/workflow-execution-v0.4.md)
- [`docs/diagrams/execution-persistence-v0.5.md`](docs/diagrams/execution-persistence-v0.5.md)

## Architecture Evolution

```text
V0.1  Foundation
  ↓
V0.2  Visual Workflow Builder
  ↓
V0.3  Workflow Validation
  ↓
V0.5  Execution Persistence   ← current
  ↓
V0.6  Asynchronous Execution
  ↓
V0.7  Worker Pool
  ↓
V0.8  Reliability
  ↓
V0.9  Idempotency
  ↓
V0.10 Scheduling
  ↓
...
V1.0 Production-Grade FlowForge
```

The project deliberately avoids premature infrastructure. RabbitMQ, Redis, workers, schedulers, WebSockets, microservices, Kubernetes, and other distributed components will be introduced only when a later version creates a concrete engineering problem that requires them.

## Next Version

**V0.6 — Asynchronous Execution**

V0.6 will address the request-lifecycle limitation of synchronous execution by introducing asynchronous execution and RabbitMQ. Workers remain a separate concern for V0.7.


## V0.6 — Asynchronous Execution

### Goal

V0.6 answers:

> Can FlowForge decouple API requests from workflow execution by durably recording an execution in PostgreSQL and dispatching execution work through RabbitMQ?

The answer is yes.

### Implemented and verified

- RabbitMQ added as the asynchronous execution transport
- Durable RabbitMQ exchange, queue, and routing key for execution jobs
- JSON message conversion through Spring AMQP/Jackson
- `ExecutionJob` message containing the persisted execution ID
- Execution records now enter `QUEUED` before asynchronous processing
- `POST /api/workflows/{workflowId}/execute` returns HTTP `202 Accepted`
- API response reduced to the stable execution identifier and current status
- Exact workflow-version lookup during worker processing
- Dedicated `ExecutionWorkerService` execution boundary
- RabbitMQ listener/consumer for queued execution jobs
- Worker loads the execution and exact persisted workflow version from PostgreSQL
- Worker reuses the existing workflow execution engine rather than creating a second execution engine
- Existing execution/node lifecycle persistence continues to record the asynchronous run
- Controller integration coverage verifies `202 QUEUED`, eventual success, and the full seven-event lifecycle
- Consumer and worker unit-test coverage
- Full backend test suite verification after the V0.6 lifecycle correction

### Asynchronous execution flow

```text
Client
  |
  | POST /api/workflows/{workflowId}/execute
  v
Spring Boot API
  |
  | create execution in PostgreSQL
  | status = QUEUED
  v
PostgreSQL
  |
  | publish ExecutionJob(executionId)
  v
RabbitMQ
  |
  | execution.created
  v
ExecutionJobConsumer
  |
  v
ExecutionWorkerService
  |
  +--> load Execution by executionId
  |
  +--> load exact WorkflowVersion
  |
  +--> load exact WorkflowDefinition snapshot
  |
  +--> mark execution RUNNING
  |
  v
WorkflowExecutionEngine
  |
  +--> Start
  +--> Transform / HTTP Request
  +--> lifecycle persistence
  |
  v
PostgreSQL
  |
  +--> EXECUTION_COMPLETED / EXECUTION_FAILED
  +--> node records
  +--> execution events
```

The API does not execute workflow nodes before returning. It creates the execution record, transitions it to `QUEUED`, publishes a small RabbitMQ job containing the execution ID, and returns the execution identifier with HTTP `202`.

The worker is intentionally thin. PostgreSQL remains the source of truth for execution state and workflow definitions; RabbitMQ only transports the work signal.

### V0.6 execution lifecycle

```text
API request
    |
    v
QUEUED
    |
    | RabbitMQ dispatch
    v
RUNNING
    |
    +--> NODE_STARTED
    |        |
    |        v
    |   node executor
    |        |
    |   +----+----+
    |   |         |
    | success   failure
    |   |         |
    |   v         v
    | NODE_      NODE_FAILED
    | COMPLETED      |
    |   |            |
    +---+------------+
        |
        +--> EXECUTION_COMPLETED -> SUCCESS
        |
        +--> EXECUTION_FAILED    -> FAILED
```

For a successful two-node workflow, the expected persisted event order is:

```text
EXECUTION_QUEUED
EXECUTION_STARTED
NODE_STARTED
NODE_COMPLETED
NODE_STARTED
NODE_COMPLETED
EXECUTION_COMPLETED
```

The terminal execution event is written by the engine's lifecycle listener. The worker does not duplicate the terminal persistence call.

### Execution API in V0.6

```text
POST /api/workflows/{workflowId}/execute
GET  /api/workflows/{workflowId}/executions
GET  /api/executions/{executionId}
```

Example request:

```json
{
  "triggerData": {
    "message": "hello"
  }
}
```

The execution request returns immediately with a response shaped like:

```json
{
  "executionId": "<uuid>",
  "status": "QUEUED"
}
```

The client can use the returned execution ID with the execution-detail endpoint to observe the persisted state.

### Why the queue message contains only an execution ID

The message intentionally does not contain a full workflow definition or mutable execution payload. The execution ID is the durable correlation key. The worker reloads the execution and exact workflow-version data from PostgreSQL before running the workflow.

This keeps PostgreSQL as the source of truth and prevents the RabbitMQ message from becoming a second authoritative copy of workflow state.

### V0.6 boundary

V0.6 adds asynchronous dispatch, not a complete distributed execution platform.

It deliberately does not include:

- worker pools or horizontal worker scaling
- retries or dead-letter queues
- idempotency / deduplication guarantees
- scheduling
- delayed execution
- WebSockets or push status updates
- transactional outbox
- exactly-once delivery semantics
- branching or parallel workflow execution
- distributed locking

A database commit and RabbitMQ publish are separate operations in V0.6. A process failure between those operations can therefore leave a persisted `QUEUED` execution without a published job. Reliability mechanisms such as an outbox belong to a later version.

### Local RabbitMQ infrastructure

Docker Compose now includes RabbitMQ with the management UI:

```text
AMQP:       localhost:5672
Management: http://localhost:15672
Username:   guest
Password:   guest
```

The backend connects to RabbitMQ using the Spring Boot AMQP configuration in `application.yml`.

### Architecture after V0.6

```text
React + TypeScript + Vite
          |
          | HTTP / REST
          v
     Spring Boot API
          |
     +----+-------------------+
     |                        |
     v                        v
PostgreSQL                RabbitMQ
(source of truth)         (async transport)
                              |
                              v
                        Execution Worker
                              |
                              v
                    WorkflowExecutionEngine
                              |
                              v
                         PostgreSQL
```

FlowForge remains a modular monolith. The worker is a logical execution boundary inside the Spring Boot application; V0.6 does not introduce an independent microservice.
