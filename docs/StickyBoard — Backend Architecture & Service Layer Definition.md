# StickyBoard — Backend Architecture & Service Layer Definition

*Last updated: 2025‑10‑17*

This document defines the **backend architecture**, **service/repository responsibilities**, and **inter-module contracts** for StickyBoard. It is **stack-agnostic**, suitable for .NET, Java, Node.js, or Python ecosystems.

------

## 1) Architectural Overview

**Goal:** modular, service‑driven backend that maps directly to REST API and PostgreSQL schema while supporting offline sync, rules, and clustering logic.

### Layered Architecture

```
Presentation Layer (REST Controllers)
│
├─ Application Layer (Services)
│    ├─ Orchestrates business logic and validation
│    └─ Calls repositories, rule engine, sync service
│
├─ Domain Layer (Models, DTOs, Mappers)
│
└─ Infrastructure Layer
     ├─ Repositories (PostgreSQL + SQLite adapters)
     ├─ Rule Engine Runtime
     ├─ Sync Engine (Operations log)
     ├─ FTS / Search Index
     └─ Storage / File Service
```

------

## 2) Core Backend Modules

### 2.1 **Auth Service**

Handles user registration, login, token generation, and refresh.

- JWT creation & validation.
- Password hashing (bcrypt/argon2).
- Role extraction for authorization.
- Token revocation & refresh management.

**Interfaces:**

- `IAuthService`
  - `register(email, password, displayName)`
  - `login(email, password)`
  - `refreshToken(token)`
  - `logout(userId)`

### 2.2 **User Service**

User profile CRUD, preferences, and admin listing.

- Update display name or preferences.
- Retrieve current user (from JWT).
- Admin listing/deletion of users.

**Interfaces:**

- `IUserService`
  - `getUser(id)`
  - `getCurrentUser(context)`
  - `updateUser(id, payload)`
  - `listUsers(filter)`
  - `deleteUser(id)` *(admin only)*

### 2.3 **Board Service**

Central orchestration for boards, sections, tabs, collaborators, and rules.

- Create/update/delete boards.
- Manage collaborators and permissions.
- Cascade deletion of sections/cards.
- Retrieve entire board hierarchy.

**Interfaces:**

- `IBoardService`
  - `createBoard(ownerId, dto)`
  - `updateBoard(boardId, dto)`
  - `deleteBoard(boardId)`
  - `getBoard(boardId)`
  - `listBoards(userId, visibility?)`
  - `addCollaborator(boardId, userId, role)`
  - `updateCollaborator(boardId, userId, role)`
  - `removeCollaborator(boardId, userId)`
  - `getCollaborators(boardId)`

### 2.4 **Section Service**

CRUD and layout updates for board sections.

- Validate board ownership or edit rights.
- Reorder sections.

**Interfaces:**

- `ISectionService`
  - `createSection(boardId, dto)`
  - `updateSection(sectionId, dto)`
  - `deleteSection(sectionId)`
  - `listSections(boardId)`

### 2.5 **Tab Service**

- Tab creation, reordering, and configuration.
- Scope enforcement (board vs section tabs).

**Interfaces:**

- `ITabService`
  - `createTab(boardId, dto)`
  - `updateTab(tabId, dto)`
  - `deleteTab(tabId)`
  - `listTabs(boardId)`

### 2.6 **Card Service**

Handles the most complex operations.

- CRUD and movement between sections/tabs.
- Version management and conflict detection.
- Tag and assignee management.
- Event emission (for rules, activities, sync).

**Interfaces:**

- `ICardService`
  - `createCard(boardId, dto)`
  - `getCard(cardId)`
  - `updateCard(cardId, dto)`
  - `deleteCard(cardId)`
  - `moveCard(cardId, sectionId, position)`
  - `addTags(cardId, tags[])`
  - `removeTag(cardId, tag)`
  - `addAssignees(cardId, users[])`
  - `removeAssignee(cardId, userId)`

**Internal Hooks:**

- Emits `CardUpdatedEvent` → RuleEngine, ActivityService, SyncService.

### 2.7 **Link Service**

Manages card-to-card relationships (dependencies, references).

**Interfaces:**

- `ILinkService`
  - `createLink(fromId, toId, type)`
  - `deleteLink(linkId)`
  - `getLinks(cardId)`

### 2.8 **Cluster Service**

Groups related cards automatically or manually.

- Builds/updates clusters.
- Manages cluster membership.
- Invokes search index for similarity.

**Interfaces:**

- `IClusterService`
  - `createCluster(boardId, type, ruleDef?)`
  - `deleteCluster(clusterId)`
  - `listClusters(boardId)`
  - `rebuildClusters(boardId)`

### 2.9 **Rule Engine Service**

Executes automation and suggestions locally or server-side.

- Evaluates triggers (`onCardCreate`, `onCardUpdate`, etc.).
- Applies actions (auto-tag, suggest convert, archive after delay).

**Interfaces:**

- `IRuleService`
  - `evaluate(event)`
  - `applyActions(boardId, actions[])`
  - `getRules(boardId)`
  - `addRule(boardId, ruleDef)`
  - `updateRule(ruleId, ruleDef)`
  - `deleteRule(ruleId)`

### 2.10 **Activity Service**

Stores and retrieves user activities (audit trail).

**Interfaces:**

- `IActivityService`
  - `record(eventType, actorId, boardId, cardId?, payload)`
  - `listActivities(boardId, since?, limit?)`

### 2.11 **File Service**

Manages file metadata and S3/minio uploads.

- Generates pre-signed upload/download URLs.
- Stores file metadata in DB.

**Interfaces:**

- `IFileService`
  - `upload(file, meta)`
  - `getFile(id)`
  - `deleteFile(id)`

### 2.12 **Search Service**

Unified full-text and metadata search.

- Combines FTS, tags, and filters.
- Ranks results by score.

**Interfaces:**

- `ISearchService`
  - `search(query, filters)`
  - `indexCard(card)`
  - `removeFromIndex(cardId)`

### 2.13 **Sync Service**

Handles operation logs, versioning, and delta exchange.

- Append operations from clients.
- Validate version conflicts.
- Push/pull deltas.

**Interfaces:**

- `ISyncService`
  - `pushOperations(userId, operations[])`
  - `getDeltas(sinceCursor, userId)`
  - `resolveConflict(op)`

### 2.14 **Admin Service**

Restricted operations for admins.

- Manage users and boards.
- Health diagnostics.

**Interfaces:**

- `IAdminService`
  - `listUsers()`
  - `listBoards()`
  - `deleteBoard(id)`
  - `getHealth()`

------

## 3) Repository Layer

Repositories handle direct database access (PostgreSQL) and persistence operations.

**Base Repository pattern:**

```
interface IRepository<T> {
  findById(id: UUID): T
  list(filter?): List<T>
  create(entity: T): T
  update(id: UUID, patch: Partial<T>): T
  delete(id: UUID): void
}
```

Specialized repositories extend for advanced queries.

### Main Repositories

| Repository            | Purpose                                     |
| --------------------- | ------------------------------------------- |
| `UserRepository`      | CRUD on users, findByEmail                  |
| `BoardRepository`     | CRUD + listByUser                           |
| `SectionRepository`   | CRUD + reorder                              |
| `TabRepository`       | CRUD + boardTabs                            |
| `CardRepository`      | CRUD + filter + byBoard + bySection + byTag |
| `TagRepository`       | upsertTags, listByBoard                     |
| `LinkRepository`      | relations between cards                     |
| `ClusterRepository`   | membership management                       |
| `RuleRepository`      | board-level rule definitions                |
| `ActivityRepository`  | append and list activities                  |
| `FileRepository`      | file metadata management                    |
| `OperationRepository` | operation log CRUD for sync                 |

------

## 4) Event Bus & Background Workers

### Event Bus

- Emits events from services: `CardUpdatedEvent`, `RuleTriggeredEvent`, `ClusterRebuildRequested`.
- Subscribed by RuleService, SearchService, ActivityService, SyncService.

### Workers

- **SearchIndexerWorker**: builds and refreshes FTS.
- **ClusterRebuilderWorker**: recomputes cluster memberships.
- **RuleExecutorWorker**: schedules delayed rules (e.g., archive after 7 days).

------

## 5) Security & Authorization Middleware

- **AuthMiddleware**: validates JWT; injects user context.
- **RoleGuard**: checks `permissions.role` before route handler.
- **RateLimiter**: enforces per-user API quotas.
- **AuditLogger**: records all admin and write operations to `activities`.

------

## 6) Sync and Version Control Workflow

1. Client performs local operations → queued.
2. On sync, client POSTs to `/api/sync`.
3. SyncService validates each op and forwards to the relevant Service (e.g., `CardService.update`).
4. Accepted ops written to `operations` table and broadcast to other clients via WebSocket.
5. Conflicts flagged → client retries after pulling latest version.

------

## 7) Extensibility Hooks

- **Plugins**: inject custom rule packs or card renderers.
- **Adapters**: replace FileService backend (local disk, S3, GCS).
- **Integrations**: external sync (Google Tasks, iCal) through `IntegrationAdapter` interface.

------

## 8) Logging & Observability

- Structured logs (JSON) per request.
- Tracing identifiers propagate via headers (`X-Request-ID`).
- Metrics: request counts, latency, DB query times.
- Error tracking through central `ErrorService`.

------

## 9) Deployment Components

| Component              | Role                                  |
| ---------------------- | ------------------------------------- |
| API Server             | Hosts REST endpoints & WebSockets     |
| Worker Service         | Executes async jobs (rules, clusters) |
| DB Server              | PostgreSQL instance                   |
| Object Storage         | Files/attachments                     |
| Cache Layer (optional) | Redis for presence, rate limits       |

------

### End of Backend Service Definition