# StickyBoard – API Plan (HTTP Service Layer)

## Overview

The StickyBoard API is a stateless HTTP service responsible for managing authentication, CRUD operations, synchronization, and offline-first consistency across devices. It uses PostgreSQL with manual SQL via `NpgsqlDataReader` and does not perform heavy background processing (delegated to Workers).

### Goals

- Provide a unified REST interface for all StickyBoard clients.
- Handle authentication, authorization, and validation.
- Maintain offline-first consistency using the `operations` table.
- Publish lightweight events for background Workers.
- Keep logic modular and transactional without blocking.

------

## 1. Architecture Layers

```
Controllers → Services → Repositories → Database
```

### Controllers

- Handle HTTP requests and responses.
- Map DTOs to models and vice versa.
- Enforce authentication and authorization.
- Record operations for synchronization.

### Services

- Contain domain logic and validation.
- Handle cross-entity coordination.
- Manage transaction boundaries.
- Orchestrate repository calls and ensure consistency.

### Repositories

- Use raw SQL with `NpgsqlDataReader` for high performance.
- Map query results to strongly-typed models.
- Provide async transactional operations.

### Common Utilities

- JWT authentication middleware.
- Global error handler.
- Operation logging utilities.
- Sync cursor generation utilities.

------

## 2. Folder Structure

```
/api
 ├── Controllers/
 ├── Services/
 ├── Repositories/
 ├── Models/
 ├── DTOs/
 ├── Auth/
 ├── Sync/
 ├── Common/
 ├── Events/
 └── Program.cs
```

------

## 3. Core Modules

| Module     | Description                       | Example Endpoints               |
| ---------- | --------------------------------- | ------------------------------- |
| Auth       | Registration, login, JWT handling | `/auth/login`, `/auth/register` |
| Users      | Profile, prefs, avatars           | `/users/:id`                    |
| Boards     | CRUD, sharing, permissions        | `/boards`, `/boards/:id`        |
| Sections   | CRUD, reordering                  | `/boards/:id/sections`          |
| Tabs       | Sub-views within boards/sections  | `/sections/:id/tabs`            |
| Cards      | CRUD, move, tag, assign           | `/boards/:id/cards`             |
| Tags       | Tag definitions                   | `/tags`                         |
| Links      | Card-to-card relations            | `/cards/:id/links`              |
| Rules      | Board-level automation rules      | `/boards/:id/rules`             |
| Clusters   | Logical card groupings            | `/boards/:id/clusters`          |
| Activities | Audit and user actions log        | `/boards/:id/activities`        |
| Files      | Uploads, metadata, linking        | `/files`                        |
| Sync       | Push/pull device operations       | `/sync/push`, `/sync/pull`      |
| Search     | Full-text search and filters      | `/search?q=...`                 |

------

## 4. Offline-First Enforcement

### API Responsibilities

- Require a valid `Authorization: Bearer <JWT>` header for all requests.
- Record every mutation as an entry in the `operations` table.
- Return incremental operation diffs via `/sync/pull`.
- Detect and reject conflicts (`409 Conflict`).
- Guarantee transactional consistency for batch writes.

### Client Responsibilities

- Maintain local SQLite storage and pending operations queue.
- Use optimistic updates and retries on network restore.
- Submit pending operations via `/sync/push`.
- Merge server changes using response payloads.

------

## 5. Sync Contract

### Push

**Endpoint:** `POST /api/sync/push`

- Accepts an array of operations from the client.
- Executes all mutations transactionally.
- Returns accepted and rejected operation IDs.

**Response:**

```json
{
  "accepted": ["op1", "op2"],
  "rejected": [
    { "id": "op3", "reason": "Conflict" }
  ]
}
```

### Pull

**Endpoint:** `GET /api/sync/pull?since=<cursor>`

- Returns all operations created since the provided cursor.
- Supports pagination with `next_cursor`.

**Response:**

```json
{
  "cursor": "2025-10-20T10:00:00Z",
  "operations": [ ... ],
  "next_cursor": "2025-10-20T12:00:00Z"
}
```

### Conflict Example

```json
{
  "error": "Conflict",
  "entity": "card",
  "id": "b2c...",
  "server_version": 5,
  "client_version": 4
}
```

------

## 6. Events (for Workers)

The API inserts lightweight event records into `activities` and `operations`, consumed later by background workers.

| Event Source         | Emitted On         | Handled By                   |
| -------------------- | ------------------ | ---------------------------- |
| Card created/updated | `/cards` endpoints | RuleExecutor, ClusterBuilder |
| Rule changed         | `/rules`           | RuleExecutor                 |
| Cluster updated      | `/clusters`        | ClusterRebuilder             |
| Board modified       | `/boards`          | ActivityLogger               |

------

## 7. Future Extraction Hooks

- Define cross-service event interfaces: `IEventBus`, `IBackgroundDispatcher`.
- Allow in-process dispatch now, replace with a broker later.
- Maintain compatibility with worker queue schema.

------

# StickyBoard – Worker Plan (Background Services)

## Overview

The Worker layer handles background and asynchronous operations outside the API. It runs as a `.NET` service within the same Docker network, consuming activity and operation events written by the API.

### Goals

- Offload long-running and compute-heavy tasks.
- Preserve fast API response times.
- Scale horizontally with dedicated worker containers.

------

## 1. Worker Types

| Worker                     | Purpose                                    | Input                                    | Output                        |
| -------------------------- | ------------------------------------------ | ---------------------------------------- | ----------------------------- |
| **RuleExecutor**           | Evaluates automation rules.                | `CardChanged` events                     | `activities`, updated `cards` |
| **ClusterRebuilder**       | Rebuilds clusters on rule or card changes. | `ClusterRebuildRequested`, `CardChanged` | `cluster_members`             |
| **SearchIndexer**          | Updates FTS indexes.                       | `CardChanged`, `BoardChanged`            | Updated `cards.tsv`           |
| **SyncCompactor**          | Compacts old operation logs.               | Scheduled task                           | Cleaned `operations` table    |
| **NotificationDispatcher** | Sends webhooks or email alerts.            | `RuleTriggered`, `DueSoon`               | `activities`, notifications   |
| **AnalyticsExporter**      | Aggregates metrics and statistics.         | Database query                           | Cached metrics, JSON export   |

------

## 2. Event Flow

**Source → Queue → Worker**

1. API inserts events in `activities` or `operations`.
2. Worker’s EventWatcher polls for new events.
3. Worker dispatches appropriate service logic.

------

## 3. Execution Model

- Each worker is implemented as a `BackgroundService`.
- Configurable via environment variables.
- Uses dependency injection for `IEventBus`, repositories, and logging.
- Handles retries and graceful shutdowns.

**Example Skeleton:**

```csharp
public class RuleExecutorWorker : BackgroundService {
  private readonly IEventBus _bus;
  private readonly IRuleService _rules;

  public RuleExecutorWorker(IEventBus bus, IRuleService rules) {
    _bus = bus; _rules = rules;
  }

  protected override Task ExecuteAsync(CancellationToken ct) {
    _bus.Subscribe<CardChanged>(async (evt, token) => {
      var actions = await _rules.EvaluateAsync(evt.BoardId, evt.CardId, token);
      await _rules.ApplyAsync(evt.BoardId, actions, token);
    });
    return Task.CompletedTask;
  }
}
```

------

## 4. Deployment Model

**Docker Compose Example:**

```yaml
services:
  api:
    build: ./StickyBoard.Api
    ports:
      - "8080:80"
    environment:
      - DATABASE_URL=postgres://...
    depends_on:
      - db

  worker:
    build: ./StickyBoard.Worker
    environment:
      - DATABASE_URL=postgres://...
    depends_on:
      - db
```

Both containers share the same Docker network and database.

------

## 5. Worker Responsibilities

| Layer    | Task                   | Details                            |
| -------- | ---------------------- | ---------------------------------- |
| Core     | Event subscription     | Poll and process domain events.    |
| Logic    | Rule and cluster logic | Execute batch or reactive jobs.    |
| Logging  | Job tracing            | Write results to `activities`.     |
| Recovery | Retry failed jobs      | Mark as failed, retry up to limit. |

------

## 6. Scaling and Future Extensions

- Migrate from polling to Redis, RabbitMQ, or NATS broker.
- Add distributed locks for concurrent job safety.
- Introduce sandboxed user-defined rules.
- Integrate vector similarity for smart clustering.
- Build dashboard for worker monitoring.

------

## 7. Summary

- **API:** Stateless, transactional, lightweight.
- **Workers:** Asynchronous, event-driven, scalable.
- **Database:** Single source of truth shared between both.

Future integrations: message broker, metrics collection, and orchestration (Quartz/Hangfire).