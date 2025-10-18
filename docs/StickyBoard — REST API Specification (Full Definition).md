# StickyBoard — REST API Specification (Full Definition)

*Last updated: 2025‑10‑17*

This document defines the complete **StickyBoard REST API** for server communication. It covers all endpoints, their HTTP methods, URL patterns, headers, request/response bodies, and standardized error codes.

All endpoints are prefixed by:

```
baseURL/api/
```

------

## 0) Common Rules

### 0.1 Authentication

All endpoints require an `Authorization` header:

```
Authorization: Bearer <JWT_TOKEN>
```

Tokens identify the authenticated user. Expired or missing tokens result in `401 Unauthorized`.

### 0.2 Content Type

Requests and responses use JSON:

```
Content-Type: application/json
Accept: application/json
```

### 0.3 Standard Response Envelope

```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

Or on failure:

```json
{
  "success": false,
  "error": {
    "code": "ERR_NOT_FOUND",
    "message": "Board not found."
  }
}
```

### 0.4 Common Error Codes

| HTTP | Code             | Message                                  |
| ---- | ---------------- | ---------------------------------------- |
| 400  | ERR_BAD_REQUEST  | Invalid request body or parameters.      |
| 401  | ERR_UNAUTHORIZED | Missing or invalid authentication token. |
| 403  | ERR_FORBIDDEN    | User lacks permission for the resource.  |
| 404  | ERR_NOT_FOUND    | Resource not found.                      |
| 409  | ERR_CONFLICT     | Version or state conflict detected.      |
| 422  | ERR_VALIDATION   | Validation error on input fields.        |
| 429  | ERR_RATE_LIMIT   | Too many requests.                       |
| 500  | ERR_INTERNAL     | Unexpected server error.                 |

------

## 1) Authentication Endpoints

### POST `/api/auth/login`

**Body:**

```json
{ "email": "user@example.com", "password": "pass1234" }
```

**Response:**

```json
{ "token": "<JWT_TOKEN>", "user": {"id":"uuid", "email":"user@example.com"} }
```

**Errors:**

- 401 Invalid credentials

### POST `/api/auth/register`

**Body:**

```json
{ "email": "user@example.com", "password": "pass1234", "displayName": "Alex" }
```

**Response:** `201 Created` → same as login.

### POST `/api/auth/refresh`

**Body:** `{ "refreshToken": "<token>" }`
 **Response:** `{ "token": "<new JWT>" }`

### POST `/api/auth/logout`

Revokes token. No body.
 **Response:** `204 No Content`

------

## 2) Users

### GET `/api/users/me`

Returns the authenticated user profile.
 **Response:**

```json
{ "id": "uuid", "email": "...", "displayName": "...", "prefs": { ... } }
```

### PATCH `/api/users/me`

**Body:** `{ "displayName": "New Name", "prefs": { ... } }`
 **Response:** updated user.

### GET `/api/users/:id`

Admin‑only access.
 **Errors:** 403 if non‑admin.

### DELETE `/api/users/:id`

Admin‑only user removal.

------

## 3) Boards

### GET `/api/boards`

List all boards for the authenticated user.
 **Query:** `?visibility=shared|private`
 **Response:** Array of board summaries.

### POST `/api/boards`

Create a new board.
 **Body:**

```json
{ "title": "Project X", "visibility": "private", "theme": {"color":"blue"} }
```

**Response:** `201 Created` → full board object.

### GET `/api/boards/:id`

Retrieve board and contained sections/tabs.

### PATCH `/api/boards/:id`

Update board metadata.
 **Body:** `{ "title": "Updated Title", "theme": {...} }`

### DELETE `/api/boards/:id`

Delete a board and its contents.
 **Response:** `204 No Content`

### GET `/api/boards/:id/collaborators`

List collaborators with roles.

### POST `/api/boards/:id/collaborators`

Add collaborator.
 **Body:** `{ "userId": "uuid", "role": "editor" }`

### PATCH `/api/boards/:id/collaborators/:userId`

Change collaborator role.

### DELETE `/api/boards/:id/collaborators/:userId`

Remove collaborator.

**Error codes:** 403 if non‑owner.

------

## 4) Sections

### GET `/api/boards/:boardId/sections`

List sections within a board.

### POST `/api/boards/:boardId/sections`

Create section.
 **Body:** `{ "title": "Backlog", "position": 1 }`

### PATCH `/api/sections/:id`

Update section title, position, or layout.

### DELETE `/api/sections/:id`

Removes section and its cards.

**Errors:** 404 if not found, 403 if no edit permission.

------

## 5) Tabs

### GET `/api/boards/:boardId/tabs`

List all tabs for a board.

### POST `/api/boards/:boardId/tabs`

Create new tab.
 **Body:** `{ "title": "Archive", "scope": "board" }`

### PATCH `/api/tabs/:id`

Update tab title or layout config.

### DELETE `/api/tabs/:id`

Remove tab.

------

## 6) Cards

### GET `/api/boards/:boardId/cards`

List cards within board (supports filters: `?section=uuid&type=task&status=open`).

### POST `/api/boards/:boardId/cards`

Create card.
 **Body:** minimal required fields per schema.

### GET `/api/cards/:id`

Fetch a single card.

### PATCH `/api/cards/:id`

Partial update.
 **Body:** any modifiable field (e.g. `title`, `status`, `content`).

### DELETE `/api/cards/:id`

Delete card.

### POST `/api/cards/:id/move`

Move card to another section/tab.
 **Body:** `{ "sectionId": "uuid", "position": 3 }`

### POST `/api/cards/:id/links`

Create link to another card.
 **Body:** `{ "toCardId": "uuid", "relType": "depends_on" }`

### DELETE `/api/links/:id`

Remove link.

### POST `/api/cards/:id/comments`

Add comment.
 **Body:** `{ "text": "Great idea!" }`

### GET `/api/cards/:id/comments`

List comments.

------

## 7) Tags & Assignees

### POST `/api/cards/:id/tags`

Add tag(s).
 **Body:** `{ "tags": ["urgent","work"] }`

### DELETE `/api/cards/:id/tags/:tag`

Remove tag.

### POST `/api/cards/:id/assignees`

Add assignee(s).
 **Body:** `{ "assignees": ["user-123"] }`

### DELETE `/api/cards/:id/assignees/:userId`

Remove assignee.

------

## 8) Rules

### GET `/api/boards/:boardId/rules`

List automation rules.

### POST `/api/boards/:boardId/rules`

Create rule.
 **Body:** rule JSON definition.

### PATCH `/api/rules/:id`

Modify rule.

### DELETE `/api/rules/:id`

Remove rule.

------

## 9) Clusters

### GET `/api/boards/:boardId/clusters`

List clusters.

### POST `/api/boards/:boardId/clusters`

Create new cluster (manual or rule‑based).

### GET `/api/clusters/:id`

Retrieve cluster details + member cards.

### DELETE `/api/clusters/:id`

Delete cluster.

------

## 10) Files (Attachments)

### POST `/api/files/upload`

Multipart/form‑data upload.
 **Headers:** `Content-Type: multipart/form-data`
 **Body fields:** `file`, `boardId`, `cardId`.
 **Response:** `{ "id": "uuid", "url": "..." }`

### GET `/api/files/:id`

Metadata for file.

### DELETE `/api/files/:id`

Delete attachment.

**Error:** 403 if not owner.

------

## 11) Activities

### GET `/api/boards/:boardId/activities`

List board activity stream.
 **Query:** `?since=timestamp&limit=50`

------

## 12) Search

### POST `/api/search`

Unified search.
 **Body:**

```json
{ "query": "meeting tomorrow", "filters": {"boardId": "uuid", "type": ["task"]} }
```

**Response:** array of results `{ type, id, title, snippet, score }`

------

## 13) Sync

### GET `/api/sync?since=<cursor>`

Pull operations since last cursor.
 **Response:**

```json
{ "cursor": "2025-10-16T10:00:00Z", "operations": [ ... ] }
```

### POST `/api/sync`

Push local operations.
 **Body:** `{ "operations": [ { ... } ] }`
 **Response:** `{ "accepted": [..], "conflicts": [..] }`

------

## 14) Admin Endpoints (Protected)

Requires `role: admin` in token.

### GET `/api/admin/users`

List all users.

### GET `/api/admin/boards`

List all boards with metadata.

### DELETE `/api/admin/boards/:id`

Force delete a board.

### GET `/api/admin/health`

Server health / uptime / version.

**Error:** 403 for non‑admins.

------

## 15) WebSocket / SSE (optional future)

Endpoint: `/api/ws`

- Authentication via JWT query param.
- Channels: `boards:{id}`, `user:{id}`.
- Events: `card.updated`, `activity.added`, `presence.update`.

------

## 16) Error Code Registry

| Code             | Description                     | Resolution              |
| ---------------- | ------------------------------- | ----------------------- |
| ERR_BAD_REQUEST  | Invalid input.                  | Correct client payload. |
| ERR_UNAUTHORIZED | Missing or invalid token.       | Login again.            |
| ERR_FORBIDDEN    | No permission.                  | Check user role.        |
| ERR_NOT_FOUND    | Entity missing.                 | Verify ID.              |
| ERR_CONFLICT     | Version mismatch or duplicate.  | Pull latest and retry.  |
| ERR_VALIDATION   | Field failed schema validation. | Fix input format.       |
| ERR_RATE_LIMIT   | Too many requests.              | Wait before retry.      |
| ERR_INTERNAL     | Unhandled server error.         | Contact support.        |

------

### End of REST API Specification