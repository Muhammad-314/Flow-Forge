# FlowForge

FlowForge is a workflow automation platform designed around a custom workflow execution engine.

The project is intentionally developed version by version. Each version solves a concrete limitation of the previous version while remaining a working product.

## Current Version

**V0.3 — Workflow Validation**

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

The backend remains a **modular monolith**. V0.2 extends the workflow module with workflow-definition persistence rather than introducing a separate service.

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

The migrations are:

```text
V1__create_users_and_workflows.sql
V2__create_workflow_definitions.sql
V3__add_start_nodes_to_workflows.sql
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

V0.3 backend verification completed successfully with:

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

### Architecture

- [`docs/architecture/v0.1-architecture.md`](docs/architecture/v0.1-architecture.md)
- [`docs/architecture/v0.2-architecture.md`](docs/architecture/v0.2-architecture.md)
- [`docs/architecture/v0.3-architecture.md`](docs/architecture/v0.3-architecture.md)

### API

- [`docs/api/workflow-definition.md`](docs/api/workflow-definition.md)

### Architecture Decision Records

- [`docs/decisions/ADR-001-java-spring-boot.md`](docs/decisions/ADR-001-java-spring-boot.md)
- [`docs/decisions/ADR-002-postgresql.md`](docs/decisions/ADR-002-postgresql.md)
- [`docs/decisions/ADR-003-modular-monolith.md`](docs/decisions/ADR-003-modular-monolith.md)
- [`docs/decisions/ADR-004-monorepo.md`](docs/decisions/ADR-004-monorepo.md)
- [`docs/decisions/ADR-005-flyway.md`](docs/decisions/ADR-005-flyway.md)

### Diagrams

- [`docs/diagrams/workflow-definition-v0.2.md`](docs/diagrams/workflow-definition-v0.2.md)
- [`docs/diagrams/workflow-definition-v0.3-validation.md`](docs/diagrams/workflow-definition-v0.3-validation.md)

## Architecture Evolution

```text
V0.1  Foundation
  ↓
V0.2  Visual Workflow Builder
  ↓
V0.3  Workflow Validation             ← current
  ↓
V0.4  Synchronous Execution Engine
  ↓
V0.5  Execution Persistence
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

**V0.4 — Synchronous Execution Engine**

V0.4 will introduce the first workflow execution path. It will execute an already-validated workflow definition synchronously within the backend request lifecycle.

The execution engine will build on the validated graph established in V0.3 rather than re-implementing structural validation.
