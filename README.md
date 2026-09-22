# FlowForge

FlowForge is a workflow automation platform designed around a custom workflow execution engine.

The project is intentionally developed version by version. Each version solves a concrete limitation of the previous version while remaining a working product.

## Current Version

**V0.9 — Idempotency**

V0.1 established the backend foundation and basic product surface. V0.2 adds the first persistent visual workflow-definition system. V0.9 is the current completed idempotency checkpoint.

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


## V0.7 — Controlled Concurrent Worker Execution

### Goal

V0.7 answers:

> Can FlowForge process multiple queued workflow executions concurrently while keeping execution state isolated and the worker model deliberately simple?

The answer is yes.

### Implemented and verified

- Configurable Spring AMQP listener concurrency
- Fixed worker-listener concurrency of **3** for the current local deployment
- `concurrency: 3` and `max-concurrency: 3` in `application.yml`
- Three active RabbitMQ consumers attached to `flowforge.execution.queue`
- Per-consumer RabbitMQ prefetch remains **250**
- Existing `ExecutionJob(UUID executionId)` message contract is unchanged
- Existing `ExecutionJobConsumer → ExecutionWorkerService → WorkflowExecutionEngine` boundary is unchanged
- No additional `ExecutorService` was introduced inside the consumer or worker
- Multiple executions can enter `ExecutionWorkerService.process(...)` concurrently
- `ExecutionContext` remains per-execution and mutable state is not shared between concurrent executions
- Executor registry remains read-only after construction and is safe for concurrent executor lookup
- Concurrent integration test uses three blocked local HTTP executions to prove that all three workers can run at the same time
- Integration test verifies that all three executions reach `RUNNING` concurrently and later complete successfully
- Integration test verifies trigger-data isolation between concurrent executions
- Full backend test suite verification: **105 tests, 0 failures, 0 errors**

### Worker model

V0.7 does **not** create three worker processes or three microservices.

There is still one Spring Boot application:

```text
                 Spring Boot application
                         |
                  RabbitMQ listener
                  concurrency = 3
                         |
             +-----------+-----------+
             |           |           |
             v           v           v
          Consumer    Consumer    Consumer
             |           |           |
             +-----------+-----------+
                         |
                 ExecutionWorkerService
                         |
                 WorkflowExecutionEngine
```

The three listener invokers are concurrent consumers within the same application process. Each consumer can process one delivered execution while the workflow engine is running.

### Concurrency and execution isolation

Each asynchronous job contains only an execution ID:

```java
public record ExecutionJob(UUID executionId) {}
```

The worker reloads the execution and exact workflow-version data from PostgreSQL before execution.

Each engine invocation creates its own `ExecutionContext`. Execution-specific variables, trigger data, node outputs, and metadata therefore remain local to that invocation rather than being stored in singleton engine fields.

The shared `WorkflowNodeExecutorRegistry` is constructed once and is not mutated during execution. Individual executors currently used by FlowForge do not store per-execution mutable state in instance fields.

### Prefetch boundary

RabbitMQ reports:

```text
consumer count:     3
prefetch per consumer: 250
```

A prefetch value of 250 does **not** mean that 750 workflow executions are running simultaneously. It controls how many messages a consumer may have delivered and not yet acknowledged. Actual active workflow execution is bounded by the listener concurrency.

V0.7 therefore keeps the existing prefetch configuration unchanged rather than treating prefetch as the worker-count setting.

### Graceful shutdown boundary

V0.7 relies on the Spring AMQP listener-container lifecycle rather than introducing a custom worker executor or shutdown framework.

The listener container is responsible for stopping its consumers during application shutdown. With the current configuration, `force-stop` is not enabled, so the listener container is not being changed to forcibly interrupt normal message processing as part of this version.

V0.7 documents the lifecycle boundary but does not claim transactional recovery, retry, or exactly-once completion semantics. Those concerns remain later reliability/idempotency work.

### V0.7 boundary

V0.7 deliberately does not include:

- separate worker processes
- microservices
- horizontal worker deployment
- retry policies
- dead-letter queues
- idempotency or deduplication
- transactional outbox
- exactly-once delivery
- distributed locking
- workflow-level parallel branching
- dynamic autoscaling
- Redis or another worker-coordination store

The goal is controlled in-process concurrency, not a distributed worker platform.

### Local verification

Start the backend:

```powershell
.\mvnw.cmd spring-boot:run
```

In another terminal, verify the RabbitMQ consumers:

```powershell
docker exec flowforge-rabbitmq rabbitmqctl list_consumers
```

Expected local state:

```text
queue_name                    ack_required  prefetch_count  active
flowforge.execution.queue     true          250             true
flowforge.execution.queue     true          250             true
flowforge.execution.queue     true          250             true
```

The V0.7 integration test provides behavioral verification by submitting three executions whose HTTP nodes block until all three requests have started. The test then releases them and verifies that all three complete successfully with isolated trigger data.


## V0.8 — Reliability

### Goal

V0.8 answers:

> Can FlowForge recover from transient workflow failures without retrying forever, while preserving durable execution state and keeping RabbitMQ failure handling explicit?

The answer is yes.

V0.8 adds reliability behavior on top of the V0.6 asynchronous execution path and V0.7 controlled concurrency model.

### Implemented and verified

- Explicit node failure classification:
  - `TRANSIENT`
  - `PERMANENT`
  - `UNKNOWN`
- HTTP failure classification:
  - `408` and `429` → transient
  - `5xx` → transient
  - other `4xx` → permanent
  - I/O failures → transient
  - invalid request configuration → permanent
- Persisted execution attempt tracking
- Persisted maximum-attempt limit
- Persisted `nextRetryAt`
- Default maximum attempts: **3**
- Exponential retry backoff:
  - attempt 1 failure → 2 seconds
  - attempt 2 failure → 4 seconds
  - further retries continue with the same exponential policy, bounded by the configured listener retry behavior
- Retry lifecycle event: `EXECUTION_RETRY_SCHEDULED`
- Attempt-aware node execution persistence so retries do not collide with previous node records
- Node timeout of **30 seconds**
- Timed-out node executions classified as transient
- Timeout cancellation/interruption handling
- Spring AMQP listener retry for explicitly retryable execution failures
- Non-retryable execution failures are not retried by the RabbitMQ listener
- Durable RabbitMQ dead-letter exchange and dead-letter queue
- Exhausted transient executions are marked `FAILED` and the original job is rejected so RabbitMQ routes it to the DLQ
- PostgreSQL remains the source of truth; RabbitMQ remains the asynchronous transport
- V0.7's three-consumer concurrency model remains unchanged
- Focused reliability tests, integration tests, retry/DLQ tests, and the full backend suite all pass
- Final full backend verification: **119 tests, 0 failures, 0 errors, 0 skipped**

### Reliability flow

```text
ExecutionJob
    |
    v
RabbitMQ execution queue
    |
    v
ExecutionWorkerService
    |
    v
mark RUNNING
(attempt increments)
    |
    v
WorkflowExecutionEngine
    |
    +--> SUCCESS
    |      |
    |      v
    |   SUCCESS
    |
    +--> TRANSIENT FAILURE
    |      |
    |      +--> attempts remain
    |      |       |
    |      |       v
    |      |   QUEUED + nextRetryAt
    |      |       |
    |      |       v
    |      |   retryable exception
    |      |       |
    |      |       v
    |      |   Rabbit listener retry
    |      |
    |      +--> max attempts reached
    |              |
    |              v
    |           FAILED
    |              |
    |              v
    |           reject / don't requeue
    |              |
    |              v
    |           RabbitMQ DLX
    |              |
    |              v
    |             DLQ
    |
    +--> PERMANENT / UNKNOWN FAILURE
           |
           v
        FAILED
           |
           v
        reject / don't requeue
```

### Attempt semantics

`Execution.attempt` is the number of actual execution attempts that have entered `RUNNING`.

An execution is created with:

```text
attempt = 0
maxAttempts = 3
status = QUEUED
```

When the worker begins an attempt:

```text
attempt 1 → RUNNING
attempt 2 → RUNNING
attempt 3 → RUNNING
```

A retry is scheduled only while `attempt < maxAttempts`.

Node persistence is keyed by:

```text
execution_id + node_id + attempt
```

This allows each retry attempt to retain its own node lifecycle records without violating uniqueness.

### Timeout boundary

Each node execution has a 30-second timeout.

A timeout:

1. cancels/interupts the node task;
2. returns a transient execution failure;
3. follows the normal retry policy if attempts remain.

The timeout is a node-execution boundary, not a global workflow deadline.

### RabbitMQ reliability topology

```text
flowforge.execution
        |
        | execution.created
        v
flowforge.execution.queue
        |
        | reject after retry policy is exhausted
        v
flowforge.execution.dlx
        |
        | execution.failed
        v
flowforge.execution.dlq
```

The DLQ is for jobs that the listener ultimately rejects without requeueing. It is not a second source of truth for execution state.

### Important reliability boundary

V0.8 does **not** claim exactly-once execution.

A workflow node can have externally visible side effects before a failure is observed. Idempotency and deduplication are therefore intentionally deferred to V0.9.

V0.8 also does not introduce a transactional outbox or durable scheduler. A future crash between a PostgreSQL state transition and message publication remains a separate reliability concern.



## V0.9 — Idempotency

### Goal

V0.9 answers:

> Can FlowForge safely tolerate duplicate RabbitMQ deliveries without executing the same persisted workflow execution more than once?

The answer is yes, within the execution-claim boundary implemented by PostgreSQL.

V0.9 builds on V0.8's at-least-once RabbitMQ and bounded-retry model. It does not attempt to turn RabbitMQ into an exactly-once transport.

### Implemented and verified

- Atomic PostgreSQL execution claim using a conditional `UPDATE`
- `QUEUED -> RUNNING` claim boundary guarded by execution status
- Attempt-limit guard during claiming
- Atomic attempt increment at the claim boundary
- `tryMarkExecutionRunning(...)` returning an explicit claim result
- Duplicate delivery after `SUCCESS` is ignored
- Duplicate delivery while `RUNNING` is ignored
- Duplicate delivery during a retry attempt is ignored
- Concurrent workers racing for the same queued execution produce exactly one successful claim
- Duplicate claims do not create duplicate execution-start processing
- Duplicate claims do not produce duplicate external HTTP execution in the tested scenarios
- Existing V0.8 retry and DLQ behavior remains intact
- PostgreSQL remains the source of truth
- RabbitMQ remains the asynchronous transport
- No second broker, distributed lock, Redis coordinator, worker service, or microservice boundary introduced
- Focused idempotency unit/integration tests
- RabbitMQ duplicate-delivery integration coverage
- Database concurrency integration coverage
- Full backend verification: **127 tests, 0 failures, 0 errors, 0 skipped**

### Idempotent execution claim

The worker does not blindly transition an execution to `RUNNING`.

Instead, PostgreSQL performs an atomic conditional update equivalent to:

```text
UPDATE executions
SET status = RUNNING,
    attempt = attempt + 1
WHERE id = executionId
  AND status = QUEUED
  AND attempt < maxAttempts
```

The affected-row count is the claim result:

```text
1 row updated -> this worker owns the execution attempt
0 rows updated -> execution is not claimable; treat as duplicate/non-runnable delivery
```

This makes the database transition itself the concurrency boundary.

### Execution state semantics

```text
new execution
    |
    v
QUEUED / attempt 0
    |
    | successful atomic claim
    v
RUNNING / attempt 1
    |
    +----------------------+
    |                      |
 success              transient failure
    |                      |
    v                      v
 SUCCESS             QUEUED / attempt 1
                           |
                           | retry claim
                           v
                      RUNNING / attempt 2
```

A retry reuses the same execution ID. The retry transition returns the execution to `QUEUED` without incrementing the attempt. The next successful claim increments the attempt when it enters `RUNNING`.

### Duplicate-delivery behavior

A duplicate `ExecutionJob` contains the same persisted execution ID.

If the original execution is already:

```text
RUNNING
SUCCESS
FAILED
```

or otherwise not claimable, the conditional database update affects zero rows and the worker returns without invoking the workflow engine.

For a duplicate that races with an original worker while the execution is `RUNNING`, only the worker that wins the `QUEUED -> RUNNING` database claim can proceed.

### Why this is not literal exactly-once execution

V0.9 establishes an idempotent execution-claim boundary for the persisted FlowForge execution.

It does **not** establish universal exactly-once side effects.

For example, an external system may receive a request before a process failure is observed. A later retry or a separate workflow execution could still produce an externally visible side effect unless that external operation is itself idempotent or otherwise coordinated.

RabbitMQ also remains an at-least-once transport. Duplicate messages can still exist; the application prevents those duplicates from becoming duplicate processing of the same claimable execution.

### Verification

V0.9 was verified with:

- duplicate-after-success integration testing;
- duplicate-while-running integration testing;
- duplicate-during-retry integration testing;
- repository-level concurrent claim testing;
- persistence-service concurrent claim testing;
- existing V0.8 retry/DLQ integration testing;
- complete backend regression suite.

Final full-suite result:

```text
Tests run: 127
Failures: 0
Errors: 0
Skipped: 0
BUILD SUCCESS
```

### V0.9 boundary

V0.9 deliberately does not add:

- literal exactly-once delivery;
- universal exactly-once external side effects;
- transactional outbox;
- durable scheduling;
- distributed locks;
- idempotency keys for arbitrary external APIs;
- automatic deduplication across different execution IDs;
- cancellation;
- horizontal worker deployment;
- workflow-level parallel branching.

The next planned version is V0.10 Scheduling.


## Frontend V0.9 Checkpoint

The frontend catch-up completed alongside the V0.9 backend checkpoint. This work connects the existing React application to the persisted asynchronous execution APIs without introducing new backend capabilities.

Implemented and manually verified:

- Workflow list, creation, editing, saving, and reopening of persisted workflow definitions
- Workflow editor integration with asynchronous execution
- `POST /api/workflows/{workflowId}/execute` handling with HTTP `202 Accepted`
- Execution history loading and refresh after execution
- Execution-detail routing through `/executions/{executionId}`
- Execution-detail loading of execution, node, and event state
- Live polling for `QUEUED` / `RUNNING` executions
- Automatic polling termination at `SUCCESS` / `FAILED`
- Lifecycle-safe cancellation of execution polling when the relevant page unmounts
- Loading, empty, success, and error states across the workflow/execution flows
- Attempt, maximum-attempt, and next-retry visibility in the frontend
- Frontend production build and lint verification
- End-to-end browser verification of workflow persistence and asynchronous execution

The frontend does not introduce WebSockets, manual retry endpoints, scheduling, or new backend execution semantics. It observes the existing REST APIs.

See [`docs/frontend-v0.9.md`](docs/frontend-v0.9.md) for the frontend checkpoint details.

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

The backend remains a **modular monolith**. V0.7 keeps API handling, RabbitMQ consumption, worker orchestration, and workflow execution within the same Spring Boot application. Worker concurrency is provided by the Spring AMQP listener container rather than by separate worker processes or services.

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

The response contains an `executionId` and its current status. Node outputs, retry state, and terminal execution state are available through the execution-detail API.

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
V5__add_execution_retry_state.sql
V6__make_execution_nodes_attempt_aware.sql
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

The current V0.9 backend verification completed successfully with:

```powershell
.\mvnw.cmd test
```

The full backend suite passed with:

```text
Tests run: 127
Failures: 0
Errors: 0
Skipped: 0
BUILD SUCCESS
```

V0.7 additionally verifies:

- three active RabbitMQ consumers
- actual concurrent execution of three workflow jobs
- simultaneous `RUNNING` state for the three executions
- isolated trigger data across concurrent executions
- successful completion of all concurrent executions
- transient retry and attempt tracking
- exponential backoff
- node timeout behavior
- dead-letter routing after retry exhaustion
- duplicate delivery after successful execution
- duplicate delivery while an execution is running
- duplicate delivery during a retry attempt
- concurrent PostgreSQL execution-claim races
- idempotent execution-claim persistence

Historical version-specific verification details remain documented below.

### Historical V0.5 verification

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


## V0.8 — Reliability

### Goal

V0.8 answers:

> Can FlowForge recover from transient workflow failures without retrying forever, while preserving durable execution state and keeping RabbitMQ failure handling explicit?

The answer is yes.

V0.8 adds reliability behavior on top of the V0.6 asynchronous execution path and V0.7 controlled concurrency model.

### Implemented and verified

- Explicit node failure classification:
  - `TRANSIENT`
  - `PERMANENT`
  - `UNKNOWN`
- HTTP failure classification:
  - `408` and `429` → transient
  - `5xx` → transient
  - other `4xx` → permanent
  - I/O failures → transient
  - invalid request configuration → permanent
- Persisted execution attempt tracking
- Persisted maximum-attempt limit
- Persisted `nextRetryAt`
- Default maximum attempts: **3**
- Exponential retry backoff:
  - attempt 1 failure → 2 seconds
  - attempt 2 failure → 4 seconds
  - further retries continue with the same exponential policy, bounded by the configured listener retry behavior
- Retry lifecycle event: `EXECUTION_RETRY_SCHEDULED`
- Attempt-aware node execution persistence so retries do not collide with previous node records
- Node timeout of **30 seconds**
- Timed-out node executions classified as transient
- Timeout cancellation/interruption handling
- Spring AMQP listener retry for explicitly retryable execution failures
- Non-retryable execution failures are not retried by the RabbitMQ listener
- Durable RabbitMQ dead-letter exchange and dead-letter queue
- Exhausted transient executions are marked `FAILED` and the original job is rejected so RabbitMQ routes it to the DLQ
- PostgreSQL remains the source of truth; RabbitMQ remains the asynchronous transport
- V0.7's three-consumer concurrency model remains unchanged
- Focused reliability tests, integration tests, retry/DLQ tests, and the full backend suite all pass
- Final full backend verification: **119 tests, 0 failures, 0 errors, 0 skipped**

### Reliability flow

```text
ExecutionJob
    |
    v
RabbitMQ execution queue
    |
    v
ExecutionWorkerService
    |
    v
mark RUNNING
(attempt increments)
    |
    v
WorkflowExecutionEngine
    |
    +--> SUCCESS
    |      |
    |      v
    |   SUCCESS
    |
    +--> TRANSIENT FAILURE
    |      |
    |      +--> attempts remain
    |      |       |
    |      |       v
    |      |   QUEUED + nextRetryAt
    |      |       |
    |      |       v
    |      |   retryable exception
    |      |       |
    |      |       v
    |      |   Rabbit listener retry
    |      |
    |      +--> max attempts reached
    |              |
    |              v
    |           FAILED
    |              |
    |              v
    |           reject / don't requeue
    |              |
    |              v
    |           RabbitMQ DLX
    |              |
    |              v
    |             DLQ
    |
    +--> PERMANENT / UNKNOWN FAILURE
           |
           v
        FAILED
           |
           v
        reject / don't requeue
```

### Attempt semantics

`Execution.attempt` is the number of actual execution attempts that have entered `RUNNING`.

An execution is created with:

```text
attempt = 0
maxAttempts = 3
status = QUEUED
```

When the worker begins an attempt:

```text
attempt 1 → RUNNING
attempt 2 → RUNNING
attempt 3 → RUNNING
```

A retry is scheduled only while `attempt < maxAttempts`.

Node persistence is keyed by:

```text
execution_id + node_id + attempt
```

This allows each retry attempt to retain its own node lifecycle records without violating uniqueness.

### Timeout boundary

Each node execution has a 30-second timeout.

A timeout:

1. cancels/interupts the node task;
2. returns a transient execution failure;
3. follows the normal retry policy if attempts remain.

The timeout is a node-execution boundary, not a global workflow deadline.

### RabbitMQ reliability topology

```text
flowforge.execution
        |
        | execution.created
        v
flowforge.execution.queue
        |
        | reject after retry policy is exhausted
        v
flowforge.execution.dlx
        |
        | execution.failed
        v
flowforge.execution.dlq
```

The DLQ is for jobs that the listener ultimately rejects without requeueing. It is not a second source of truth for execution state.

### Important reliability boundary

V0.8 does **not** claim exactly-once execution.

A workflow node can have externally visible side effects before a failure is observed. Idempotency and deduplication are therefore intentionally deferred to V0.9.

V0.8 also does not introduce a transactional outbox or durable scheduler. A future crash between a PostgreSQL state transition and message publication remains a separate reliability concern.


## Architecture

- [`docs/architecture/v0.1-architecture.md`](docs/architecture/v0.1-architecture.md)
- [`docs/architecture/v0.2-architecture.md`](docs/architecture/v0.2-architecture.md)
- [`docs/architecture/v0.3-architecture.md`](docs/architecture/v0.3-architecture.md)
- [`docs/architecture/v0.4-architecture.md`](docs/architecture/v0.4-architecture.md)
- [`docs/architecture/v0.5-architecture.md`](docs/architecture/v0.5-architecture.md)
- [`docs/architecture/v0.6-architecture.md`](docs/architecture/v0.6-architecture.md)
- [`docs/architecture/v0.7-architecture.md`](docs/architecture/v0.7-architecture.md)
- [`docs/architecture/v0.8-architecture.md`](docs/architecture/v0.8-architecture.md)
- [`docs/architecture/v0.9-architecture.md`](docs/architecture/v0.9-architecture.md)

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
- [`docs/decisions/ADR-009-rabbitmq-async-execution.md`](docs/decisions/ADR-009-rabbitmq-async-execution.md)
- [`docs/decisions/ADR-010-controlled-concurrent-worker-execution.md`](docs/decisions/ADR-010-controlled-concurrent-worker-execution.md)
- [`docs/decisions/ADR-011-execution-reliability.md`](docs/decisions/ADR-011-execution-reliability.md)
- [`docs/decisions/ADR-012-execution-idempotency.md`](docs/decisions/ADR-012-execution-idempotency.md)

### Diagrams

- [`docs/diagrams/workflow-definition-v0.2.md`](docs/diagrams/workflow-definition-v0.2.md)
- [`docs/diagrams/workflow-definition-v0.3-validation.md`](docs/diagrams/workflow-definition-v0.3-validation.md)
- [`docs/diagrams/workflow-execution-v0.4.md`](docs/diagrams/workflow-execution-v0.4.md)
- [`docs/diagrams/execution-persistence-v0.5.md`](docs/diagrams/execution-persistence-v0.5.md)
- [`docs/diagrams/async-execution-v0.6.md`](docs/diagrams/async-execution-v0.6.md)
- [`docs/diagrams/worker-concurrency-v0.7.md`](docs/diagrams/worker-concurrency-v0.7.md)
- [`docs/diagrams/reliability-v0.8.md`](docs/diagrams/reliability-v0.8.md)
- [`docs/diagrams/idempotency-v0.9.md`](docs/diagrams/idempotency-v0.9.md)

## Architecture Evolution

```text
V0.1  Foundation
  ↓
V0.2  Visual Workflow Builder
  ↓
V0.3  Workflow Validation
  ↓
V0.4  Synchronous Execution Engine
  ↓
V0.5  Execution Persistence
  ↓
V0.6  Asynchronous Execution
  ↓
V0.7  Controlled Concurrent Worker Execution
  ↓
V0.8  Reliability
  ↓
V0.9  Idempotency   ← current
  ↓
V0.10 Scheduling
  ↓
...
V1.0 Production-Grade FlowForge
```

The project deliberately avoids premature infrastructure. RabbitMQ, Redis, worker pools, schedulers, WebSockets, microservices, Kubernetes, and other distributed components are introduced only when a concrete engineering problem justifies them.

## Next Version

**V0.10 — Scheduling**

V0.10 will address durable workflow scheduling. It remains separate from V0.9's duplicate-delivery and duplicate-execution protection.
