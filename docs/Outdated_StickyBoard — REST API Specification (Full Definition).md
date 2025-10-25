# StickyBoard — REST API Specification (v2, 2025-10-20)

All endpoints are prefixed with:

```
baseURL/api/
```

------

## 0) Common Rules

### 0.1 Authentication

All requests (except `/auth/*`) require:

```
Authorization: Bearer <JWT_TOKEN>
```

Invalid or expired tokens result in `401 Unauthorized`.

### 0.2 Headers

| Header         | Purpose                         |
| -------------- | ------------------------------- |
| `Content-Type` | `application/json`              |
| `Accept`       | `application/json`              |
| `Device-Id`    | Required for sync operations    |
| `X-Version`    | Optional optimistic concurrency |

### 0.3 Response Envelope

```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

Or on error:

```json
{
  "success": false,
  "error": { "code": "ERR_NOT_FOUND", "message": "Board not found." }
}
```

### 0.4 Standard Errors

| HTTP | Code             | Meaning                    |
| ---- | ---------------- | -------------------------- |
| 400  | ERR_BAD_REQUEST  | Invalid JSON or parameters |
| 401  | ERR_UNAUTHORIZED | Invalid/missing token      |
| 403  | ERR_FORBIDDEN    | Insufficient permissions   |
| 404  | ERR_NOT_FOUND    | Entity missing             |
| 409  | ERR_CONFLICT     | Version mismatch           |
| 422  | ERR_VALIDATION   | Field validation failed    |
| 500  | ERR_INTERNAL     | Unexpected server error    |

------

## 1) Authentication Endpoints

### POST `/api/auth/login`

Authenticate and receive a JWT.

```json
{ "email": "user@example.com", "password": "secret" }
```

**Response**: `{ "token": "<jwt>", "user": {...} }`

### POST `/api/auth/register`

Create a new user.

```json
{ "email": "user@example.com", "password": "secret", "displayName": "Alex" }
```

**Response:** `201 Created`

### POST `/api/auth/refresh`

Refresh token.

```json
{ "refreshToken": "<token>" }
```

**Response:** `{ "token": "<new_jwt>" }`

### POST `/api/auth/logout`

Invalidate the current token.
 **Response:** `204 No Content`

------

## 2) Users

### GET `/api/users/me`

Retrieve the authenticated user's profile.

### PATCH `/api/users/me`

Update display name or preferences.

```json
{ "displayName": "New Name", "prefs": { ... } }
```

**Response:** updated user object.

### GET `/api/users/:id`

Admin-only access to view a user.

### PUT `/api/users/:id`

Admin-only update of user (role, prefs).

### DELETE `/api/users/:id`

Admin-only deletion of a user.

------

## 3) Boards

### GET `/api/boards`

List all boards visible to the user.

### POST `/api/boards`

Create a new board.

```json
{ "title": "Project X", "visibility": "private", "theme": {"color": "blue"} }
```

**Response:** `201 Created`

### GET `/api/boards/:id`

Retrieve a specific board and nested sections/tabs.

### PATCH `/api/boards/:id`

Update board metadata.

```json
{ "title": "Updated Title", "theme": {...} }
```

### PUT `/api/boards/:id`

Full replace of board resource.

### DELETE `/api/boards/:id`

Delete a board and its contents.
 **Response:** `204 No Content`

### Collaborator Management

- **GET** `/api/boards/:id/collaborators`
- **POST** `/api/boards/:id/collaborators`
- **PATCH** `/api/boards/:id/collaborators/:userId`
- **DELETE** `/api/boards/:id/collaborators/:userId`

All map to `permissions` table operations.

------

## 4) Sections

### GET `/api/boards/:boardId/sections`

List all sections in a board.

### POST `/api/boards/:boardId/sections`

Create a section.

```json
{ "title": "Backlog", "position": 1 }
```

### PATCH `/api/sections/:id`

Partial update of title, position, or layout.

### PUT `/api/sections/:id`

Full section replace.

### DELETE `/api/sections/:id`

Remove a section and its cards.

------

## 5) Tabs

### GET `/api/boards/:boardId/tabs`

List all tabs in the board.

### POST `/api/boards/:boardId/tabs`

Create new tab.

```json
{ "title": "Archive", "scope": "board" }
```

### PATCH `/api/tabs/:id`

Update tab title or layout.

### PUT `/api/tabs/:id`

Replace tab definition.

### DELETE `/api/tabs/:id`

Remove a tab.

------

## 6) Cards

### GET `/api/boards/:boardId/cards`

List cards (filterable by `section`, `status`, `type`).

### POST `/api/boards/:boardId/cards`

Create new card.

```json
{ "title": "Fix bug", "type": "task", "sectionId": "..." }
```

### GET `/api/cards/:id`

Fetch one card.

### PATCH `/api/cards/:id`

Partial update (title, content, status).

### PUT `/api/cards/:id`

Replace card fully.

### DELETE `/api/cards/:id`

Remove card.

### POST `/api/cards/:id/move`

Move a card to another section/tab.

```json
{ "sectionId": "uuid", "position": 3 }
```

------

## 7) Tags & Assignees

### POST `/api/cards/:id/tags`

Add or create tags.

### DELETE `/api/cards/:id/tags/:tag`

Remove a tag from a card.

### POST `/api/cards/:id/assignees`

Assign one or multiple users.

### DELETE `/api/cards/:id/assignees/:userId`

Unassign user.

------

## 8) Rules

### GET `/api/boards/:boardId/rules`

List rules.

### POST `/api/boards/:boardId/rules`

Create rule definition.

### PATCH `/api/rules/:id`

Update part of a rule.

### PUT `/api/rules/:id`

Replace full rule definition.

### DELETE `/api/rules/:id`

Delete rule.

------

## 9) Clusters

### GET `/api/boards/:boardId/clusters`

List clusters for a board.

### POST `/api/boards/:boardId/clusters`

Create manual or rule-based cluster.

### GET `/api/clusters/:id`

Get cluster detail and members.

### PATCH `/api/clusters/:id`

Update cluster metadata.

### DELETE `/api/clusters/:id`

Remove cluster.

------

## 10) Files (Attachments)

### POST `/api/files/upload`

Upload file (multipart).

### GET `/api/files/:id`

Retrieve file metadata.

### PATCH `/api/files/:id`

Update file metadata.

### PUT `/api/files/:id`

Replace file entry (rarely used).

### DELETE `/api/files/:id`

Delete file.

------

## 11) Activities

### GET `/api/boards/:boardId/activities`

List activities by board.

### DELETE `/api/activities/:id`

(Admin) Remove log entry.

------

## 12) Search

### POST `/api/search`

Search cards and boards by text query.

```json
{ "query": "meeting tomorrow", "filters": {"boardId": "uuid"} }
```

### GET `/api/search?q=<term>`

Simple query variant.

------

## 13) Sync

### GET `/api/sync?since=<cursor>`

Pull operations since timestamp.

### POST `/api/sync`

Push local operations.

### PUT `/api/sync/resolve`

Submit conflict resolution payload.

------

## 14) Admin

Supports all CRUD operations for global management.

- **GET /api/admin/users**
- **POST /api/admin/users**
- **PUT /api/admin/users/:id**
- **DELETE /api/admin/users/:id**
- **GET /api/admin/boards**
- **DELETE /api/admin/boards/:id**

------

# StickyBoard — Worker Service Specification (2025-10-20)

## Overview

Workers execute background jobs created by the API layer through the `activities` and `worker_jobs` tables.

------

## 1) Worker Endpoints (Internal)

| Method     | Endpoint               | Description                    |
| ---------- | ---------------------- | ------------------------------ |
| **GET**    | `/api/worker/jobs`     | List queued jobs (admin view). |
| **POST**   | `/api/worker/jobs`     | Manually enqueue a job.        |
| **GET**    | `/api/worker/jobs/:id` | Inspect job details.           |
| **PATCH**  | `/api/worker/jobs/:id` | Update job status or priority. |
| **PUT**    | `/api/worker/jobs/:id` | Replace job payload.           |
| **DELETE** | `/api/worker/jobs/:id` | Cancel or delete job.          |

------

## 2) Job Management

| Method   | Endpoint                              | Purpose                 |
| -------- | ------------------------------------- | ----------------------- |
| **GET**  | `/api/worker/attempts`                | View execution history. |
| **GET**  | `/api/worker/deadletters`             | View failed jobs.       |
| **POST** | `/api/worker/deadletters/requeue/:id` | Requeue a dead job.     |

------

## 3) Supported Job Kinds

| Job Kind               | Table Enum | Description                          |
| ---------------------- | ---------- | ------------------------------------ |
| RuleExecutor           | `job_kind` | Applies rules based on card changes. |
| ClusterRebuilder       | `job_kind` | Recomputes card clusters.            |
| SearchIndexer          | `job_kind` | Updates text search vectors.         |
| SyncCompactor          | `job_kind` | Deduplicates operations.             |
| NotificationDispatcher | `job_kind` | Sends alerts/emails.                 |
| AnalyticsExporter      | `job_kind` | Aggregates metrics.                  |
| Generic                | `job_kind` | Miscellaneous task handler.          |

------

## 4) Job Lifecycle

- **queued** → waiting to be claimed.
- **running** → in progress by worker.
- **succeeded** → completed successfully.
- **failed** → retried until max_attempts.
- **dead** → archived to `worker_job_deadletters`.

------

## 5) Example Job Payload

```json
{
  "activity_id": "uuid",
  "board_id": "uuid",
  "card_id": "uuid",
  "act_type": "card_updated",
  "payload": { "before": {...}, "after": {...} }
}
```

------

## 6) Worker CRUD Summary

| HTTP   | Endpoint                            | Description                    |
| ------ | ----------------------------------- | ------------------------------ |
| GET    | /api/worker/jobs                    | Retrieve job queue.            |
| POST   | /api/worker/jobs                    | Create job manually.           |
| GET    | /api/worker/jobs/:id                | Inspect job.                   |
| PUT    | /api/worker/jobs/:id                | Replace job data.              |
| PATCH  | /api/worker/jobs/:id                | Modify job status or priority. |
| DELETE | /api/worker/jobs/:id                | Cancel job.                    |
| GET    | /api/worker/attempts                | List job attempts.             |
| GET    | /api/worker/deadletters             | List dead jobs.                |
| POST   | /api/worker/deadletters/requeue/:id | Requeue job.                   |

------

## 7) Monitoring

Workers expose `/api/admin/workers` for health and throughput stats.
 Each entry reports worker kind, claimed jobs, success/failure count, and last activity timestamp.

------

**End of StickyBoard API + Worker Documentation (2025-10-20)**