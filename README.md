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

