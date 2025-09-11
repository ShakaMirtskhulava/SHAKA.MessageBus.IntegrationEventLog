# Event-Driven Repository Abstraction

This project defines an abstraction for working with databases in an Event-Driven System.

## Overview

### IEntity Abstraction
The core of this project starts with defining a generic `IEntity` abstraction. This interface mandates that all entities must implement an `Id` property. The `Id` type is specified using a generic parameter, allowing flexibility in choosing different types of identifiers, such as `int`, `Guid`, `string`, etc., as long as the type implements `IEquatable<T>`.

### IIntegrationEventLog
The `IIntegrationEventLog` interface defines the structure of an integration event log. Since different infrastructures may require distinct implementations, this is designed as an interface rather than a class. For example, in an EF Core implementation, attributes may need to be added to its properties to align with entity framework conventions.

### Failed Message Handling
We introduce two key abstractions for handling failed messages:
- **`IFailedMessageChain`**: Manages sequentially failed messages associated with a single entity. It includes properties such as:
  - `EntityId`: Specifies the ID of the entity to which the failed messages belong.
  - `ShouldRepublish`: Determines whether the failed message republisher should retry processing the chained messages.
- **`IFailedMessage`**: Represents an individual failed message and contains:
  - The serialized failed message.
  - Error message and stack trace.
  - `ShouldSkip`: Indicates whether the message should be republished.

> **Note:** The `EventStateEnum.PublishedFailed` state is distinct from failed messages. `PublishedFailed` indicates that a message did not reach the broker, whereas failed messages occur when a handler throws an exception while processing the message.

## Services

### IIntegrationEventLogService
This service provides an abstraction for handling integration event logs, allowing us to implement the Outbox Pattern to maintain consistency across microservices. The service includes:
- **Retrieving batched pending events** for the publisher.
- **Creating failed message chains** via `AddInFailedMessageChain`.
- **Updating event statuses** through:
  - `MarkEventAsPublished()`
  - `MarkEventAsInProgress()`
  - `MarkEventAsFailed()`

### IIntegrationEventService
This service is responsible for managing actual integration events and provides:
- **Add, Update, and Remove** methods to perform CRUD operations on entities while maintaining respective integration event logs.

## Publisher Background Service
The project includes a **Publisher Background Service** that automates event publishing. It:
- Publishes pending events.
- Republishes failed events.

### Configuration
The service can be configured via `PublisherOptions`, where you can specify:
- `Delay`
- `EventsBatchSize`
- `FailedMessageChainBatchSize`
- `EventTypesAssemblyName`

To configure the Publisher Background Service, use the `ConfigurePublisher()` extension method.

---

This abstraction ensures robust event-driven database operations while preventing inconsistencies between microservices.



### Problems That Arise During Horizontal Scaling of the Orchestrator

#### 1. In‑Memory Circuit Breaker State Divergence
- **Risk:** Each node keeps its own breaker state; resumes on different nodes may bypass open circuits.
- **Fix:** 
  - Persist breaker state in a shared store (DB/Redis) keyed by dependency/step.
  - Expose `ICircuitStateStore` (`Get/Set/CompareExchange`, `OpenUntil`, success/failure counters).
  - Base "open until" on server/DB time; add TTL and fencing token.

#### 2. ResumeWorkflow Races and Duplicate Resumes
- **Risk:** Redeliveries or handler retries call `ResumeWorkflow` concurrently on different nodes.
- **Fix:** 
  - Add optimistic concurrency (rowversion/timestamp) to `WorkflowInstanceEntity`.
  - Predicate updates: `WHERE Id = @id AND State = 'Suspended' AND CurrentStepIndex = @idx`.
  - Validate against `ExpectedEventTypes`; ignore unexpected events.
  - Ensure resume is idempotent: if not suspended or step advanced, no-op.

#### 3. Duplicate Compensations
- **Risk:** Concurrent failure paths lead to compensation twice.
- **Fix:** 
  - Execute compensation only after an atomic state transition to `Compensating` guarded by concurrency token.
  - Make compensations idempotent (idempotency keys per step + instance).
  - Record `Compensation_{Step}_Done` flag and check before executing.

#### 4. Outbox Publisher Contention
- **Risk:** Multiple publishers read the same rows.
- **Fix:** 
  - Guarded claim: `UPDATE ... SET State = InProgress WHERE Id=@id AND State IN (NotPublished, Failed)` and check rows affected.
  - Prefer atomic "claim N" pattern (`UPDATE TOP(@n)... OUTPUT inserted.*`) to avoid two‑phase read/claim.
  - Keep idempotent publish on the consumer side using `EventId`.

#### 5. Auto Response Handlers on Every Node
- **Risk:** Misconfigured topology (multiple queues) causes duplicate deliveries.
- **Fix:** 
  - Ensure one shared queue per response event type (competing consumers).
  - Keep manual handlers only for non‑workflow domain events; response handlers are auto‑registered once via a singleton registry.

#### 6. Retry/Backoff Timing Skew
- **Risk:** Different clocks produce different retry due times.
- **Fix:** 
  - Compute backoff due times with DB/server time (e.g., `GETUTCDATE()`).
  - Persist `NextRetryUtc`; only resume when `now >= NextRetryUtc` with predicate updates.

#### 7. Step Execution Duplication
- **Risk:** Two nodes execute the same step due to races.
- **Fix:** 
  - Transition instance to `Running` with a predicate (state/index) before action.
  - Record `StepHistory` with a unique constraint (`InstanceId`, `StepIndex`) to reject duplicates.
  - Make step side‑effects idempotent where possible.

#### 8. Event Handler Idempotency
- **Risk:** Broker redeliveries reprocess response events.
- **Fix:** 
  - Store processed `EventIds` (per handler) with TTL; ignore duplicates.
  - Alternatively, ensure `ResumeWorkflow` is idempotent (see #2).

#### 9. Leader‑Only Orchestration Work (Future Timers/Scans)
- **Risk:** If you add due‑step scans, cleanups, or reminder jobs, each node will run them.
- **Fix:** 
  - Add leader election (DB lease, Redis lock, or K8s Lease):
    - Lease row: `OwnerId`, `LeaseId`, `ExpiresAt`, `Version`; renew heartbeats; use fencing token on writes.
    - Only leader runs scheduled scans/cleanup; followers handle events.

#### 10. Circuit Breaker Backoff Across Nodes
- **Risk:** A breaker opens on one node but another resumes too early.
- **Fix:** 
  - Store `OpenedAt`/`OpenUntil` in shared store; check `IsExecutionAllowed` against shared state.
  - Record success/failure to the same store; use monotonic version to serialize transitions.

#### 11. Clock Skew and TTLs
- **Risk:** Expiry calculations differ per node.
- **Fix:** 
  - Always use DB time for lease/expiry and retry computations.
  - Avoid relying on local system time for orchestration decisions.

#### 12. Observability Gaps
- **Risk:** Hard to diagnose split‑brain/duplication.
- **Fix:** 
  - Emit metrics: resumes attempted/succeeded, rejected resumes, duplicate compensations blocked, breaker opens, outbox claims/publishes.
  - Structured logs with `InstanceId`, `StepIndex`, `FencingToken`/`Version`.

#### 13. Queue Topology Drift
- **Risk:** Switching from direct single queue to per‑instance queues introduces duplication.
- **Fix:** 
  - Document and enforce exchange/queue bindings in code/config.
  - Add a health check to verify expected bindings at startup.