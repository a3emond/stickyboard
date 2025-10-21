# StickyBoard – DTO Reference & Usage Guide

**Purpose:**
 This guide explains each DTO, what layer uses it, and its role in the API flow.
 DTOs live in `StickyBoard.Api/DTOs` and act as the **public-facing data contract** between the API and all client apps.

------

## 1. Authentication & User DTOs

### `AuthRequestDto`, `RegisterRequestDto`

**Purpose:**
 Input payloads for login and registration endpoints.

**Used by:**
 `AuthController → AuthService`

**Endpoints:**

- `POST /auth/login`
- `POST /auth/register`

**Flow:**
 Client → DTO → Service → Repository (validate user, hash password, issue JWT)

------

### `AuthResponseDto`

**Purpose:**
 Output payload for login/registration responses, containing the JWT token.

**Used by:**
 `AuthController` (response to clients)

**Flow:**
 Repository → Entity → DTO → Controller → Client

------

### `UserDto`

**Purpose:**
 Exposes only safe user information (no password, no internal flags).

**Used by:**
 `UsersController`, `BoardsController` (to include owner/assignee display info)

**Endpoints:**

- `GET /users/:id`
- `GET /boards/:id/members`

------

## 2. Board & Permission DTOs

### `BoardDto`

**Purpose:**
 Represents a board with minimal configuration data visible to clients.

**Used by:**
 `BoardsController → BoardService`

**Endpoints:**

- `GET /boards`
- `GET /boards/:id`
- `POST /boards`

------

### `BoardCreateDto`, `BoardUpdateDto`

**Purpose:**
 Input payloads for creating or updating boards.

**Used by:**
 `BoardsController` (body → service)

**Flow:**
 Client JSON → DTO → Service → Repository → Entity → Save → Response DTO

------

### `PermissionDto`

**Purpose:**
 Represents a permission entry (user role on a board).

**Used by:**
 `BoardsController`, `PermissionsService`

**Endpoints:**

- `GET /boards/:id/permissions`
- `POST /boards/:id/permissions`

------

## 3. Section & Tab DTOs

### `SectionDto`

**Purpose:**
 Represents a visual grouping of cards inside a board.

**Used by:**
 `SectionsController`, `BoardService`

**Endpoints:**

- `GET /boards/:id/sections`
- `POST /boards/:id/sections`

------

### `TabDto`

**Purpose:**
 Represents a “view” or “context” within a board or section (calendar tab, list tab, etc.).

**Used by:**
 `TabsController`

**Endpoints:**

- `GET /sections/:id/tabs`
- `POST /sections/:id/tabs`

------

## 4. Card DTOs

### `CardDto`

**Purpose:**
 Main representation of a note, task, or event.
 Used for all card reads and sync payloads.

**Used by:**
 `CardsController`, `SyncService`, `RuleExecutorWorker`

**Endpoints:**

- `GET /boards/:id/cards`
- `POST /boards/:id/cards`
- `PATCH /cards/:id`

------

### `CardCreateDto`, `CardUpdateDto`

**Purpose:**
 Input for creating/updating cards.

**Used by:**
 `CardsController → CardService → CardRepository`

**Endpoints:**

- `POST /boards/:id/cards`
- `PATCH /cards/:id`

------

## 5. Tags, Links, and Assignees DTOs

### `TagDto`

**Purpose:**
 Represents a global tag definition (unique name).

**Used by:**
 `TagsController → TagService`

**Endpoints:**

- `GET /tags`
- `POST /tags`

------

### `CardTagDto`

**Purpose:**
 Associates a tag with a specific card.

**Used by:**
 `CardsController`, `CardTagRepository`

**Endpoints:**

- `POST /cards/:id/tags`
- `DELETE /cards/:id/tags/:tag`

------

### `LinkDto`

**Purpose:**
 Represents a relationship between two cards (dependencies, references, etc.).

**Used by:**
 `LinksController`, `RuleExecutorWorker`

**Endpoints:**

- `GET /cards/:id/links`
- `POST /cards/:id/links`

------

## 6. Clusters & Rules DTOs

### `ClusterDto`

**Purpose:**
 Represents a logical grouping of cards (manual or rule-based).

**Used by:**
 `ClustersController`, `ClusterRebuilderWorker`

**Endpoints:**

- `GET /boards/:id/clusters`
- `POST /boards/:id/clusters`

------

### `ClusterMemberDto`

**Purpose:**
 Represents a card membership inside a cluster.

**Used by:**
 `ClusterService` (internal, not exposed directly)

------

### `RuleDto`, `RuleCreateDto`

**Purpose:**
 Represents board automation rules (conditions/actions).

**Used by:**
 `RulesController`, `RuleExecutorWorker`

**Endpoints:**

- `GET /boards/:id/rules`
- `POST /boards/:id/rules`

------

## 7. Activities & Files DTOs

### `ActivityDto`

**Purpose:**
 Represents a log entry for user or system actions.

**Used by:**
 `ActivitiesController`, `ActivityLoggerWorker`

**Endpoints:**

- `GET /boards/:id/activities`

------

### `FileDto`, `FileUploadRequestDto`

**Purpose:**
 Expose file metadata and upload requests.

**Used by:**
 `FilesController`, `FileService`

**Endpoints:**

- `POST /files` (register upload)
- `GET /files/:id`

**Special:**
 May include a signed URL for upload/download.

------

## 8. Sync DTOs

### `OperationDto`

**Purpose:**
 Core sync unit — represents one local operation (create/update/delete).

**Used by:**
 `SyncService`, `SyncController`, and Workers

**Endpoints:**

- `POST /sync/push`
- `GET /sync/pull`

------

### `SyncPushRequestDto`, `SyncPushResponseDto`, `SyncPullResponseDto`

**Purpose:**
 Encapsulate request/response format for the offline-first synchronization flow.

**Used by:**
 `SyncController` only.

**Flow:**

1. Client pushes pending operations (`SyncPushRequestDto`).
2. API applies them and responds (`SyncPushResponseDto`).
3. Client pulls new remote ops (`SyncPullResponseDto`).

------

## 9. Admin & Search DTOs

### `AdminUserSummaryDto`, `AdminBoardSummaryDto`

**Purpose:**
 Used in admin dashboards for managing users/boards.

**Used by:**
 `AdminController`, `AdminService`

**Endpoints:**

- `GET /admin/users`
- `GET /admin/boards`

------

### `HealthCheckDto`

**Purpose:**
 Internal health endpoint to verify API status and version.

**Used by:**
 `AdminController` (`GET /admin/health`)

------

### `SearchRequestDto`, `SearchResultDto`

**Purpose:**
 Handles full-text search queries and results.

**Used by:**
 `SearchController → SearchService`

**Endpoints:**

- `GET /search?q=...`

------

## 10. Utility DTOs

### `IdResponseDto`

**Purpose:**
 Generic response when an operation returns only a new resource ID.
 Used for POST success responses.

------

### `ApiResponseDto<T>`

**Purpose:**
 Wraps success/error payloads in a consistent format.
 Used globally by all controllers for standard responses.

------

### `ErrorDto`

**Purpose:**
 Represents a structured API error (`code`, `message`).
 Returned when validation or permission checks fail.

------

## 11. Where DTOs Fit in the Architecture

```
Client (JSON)
   │
   ▼
Controller → (deserialize) → DTO (input)
   │
   ▼
Service → Entity → Repository → Database
   │
   ▼
Service → DTO (output)
   │
   ▼
Controller → (serialize) → JSON Response
```

**Key takeaways:**

- **DTOs define API contracts** (not tied to database).
- **Entities define persistence structure** (mapped to PostgreSQL).
- **Services handle conversions** between them.
- **Controllers** never touch entities directly — only DTOs.