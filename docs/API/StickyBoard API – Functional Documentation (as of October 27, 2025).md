# **StickyBoard API – Functional Documentation (as of October 27, 2025)**

## **1. Architecture Overview**

**Layered design:**

```
Controller  →  Service  →  Repository  →  Database (PostgreSQL via Npgsql)
```

**Key principles:**

- Strict separation of concerns.
- All operations are fully asynchronous and cancellation-aware.
- Authentication is JWT-based (`Authorization: Bearer <token>`).
- Enum types are PostgreSQL-mapped (`MapEnum<EnumName>()` in Program.cs).
- Every request operates within a scoped DI lifetime (one connection context per request).

------

## **2. Core Modules**

### **2.1 Authentication & Users**

| Component                               | Purpose                                                      |
| --------------------------------------- | ------------------------------------------------------------ |
| **AuthService**                         | Handles login, registration, token refresh, and password validation. |
| **UserService**                         | Provides user CRUD, profile fetching, and search.            |
| **JwtTokenService**                     | Issues access and refresh tokens with configurable lifetimes. |
| **RefreshTokenRepository**              | Stores refresh tokens and expiration timestamps.             |
| **AuthUserRepository / UserRepository** | Database layer for user credentials and profile data.        |

**Controllers**

- `AuthController`
  - `POST /api/auth/login` — Login and receive tokens.
  - `POST /api/auth/register` — Create a new account.
  - `POST /api/auth/refresh` — Refresh an access token.
- `UsersController`
  - `GET /api/users/{id}` — Get user profile.
  - `GET /api/users/search?query=...` — Search users.
  - `PUT /api/users/{id}` — Update profile.

**Security**

- Uses `User.GetUserId()` extension to extract `sub` claim.
- `Authorize` attribute enforces token validation per endpoint.

------

### **2.2 Collaboration Layer**

| Component               | Purpose                                     |
| ----------------------- | ------------------------------------------- |
| **MessageService**      | Handles private and system messages.        |
| **InviteService**       | Manages board and organization invitations. |
| **UserRelationService** | Manages friendships and relation statuses.  |

**Repositories**

- `MessageRepository`
- `InviteRepository`
- `UserRelationRepository`

**Controllers**

- `MessagesController`
  - `GET /api/messages` — Retrieve inbox for logged-in user.
  - `GET /api/messages/unread-count` — Count unread messages.
  - `POST /api/messages` — Send a message.
  - `PUT /api/messages/{id}/status` — Update message read status.
  - `DELETE /api/messages/{id}` — Delete a message.
- `InvitesController`
  - `POST /api/invites` — Send invite.
  - `GET /api/invites/pending` — List pending invites.
  - `PUT /api/invites/{id}/accept` — Accept invite.
  - `DELETE /api/invites/{id}` — Decline or revoke invite.
- `UserRelationsController`
  - `GET /api/relations` — List relations.
  - `POST /api/relations` — Send friend request.
  - `PUT /api/relations/{id}/status` — Update relation status.
  - `DELETE /api/relations/{id}` — Remove relation.

**Enums**

- `MessageType` — direct, system, invite.
- `RelationStatus` — pending, accepted, blocked.

------

### **2.3 Boards & Permissions**

| Component             | Purpose                                        |
| --------------------- | ---------------------------------------------- |
| **BoardService**      | Core CRUD for boards and visibility.           |
| **PermissionService** | Handles collaborator roles and access control. |

**Repositories**

- `BoardRepository`
- `PermissionRepository`

**Controllers**

- `BoardsController`
  - `GET /api/boards` — List accessible boards.
  - `POST /api/boards` — Create a new board.
  - `PUT /api/boards/{id}` — Update board metadata.
  - `DELETE /api/boards/{id}` — Delete board.
- `PermissionsController`
  - `GET /api/boards/{boardId}/permissions` — List collaborators.
  - `POST /api/boards/{boardId}/permissions` — Add collaborator.
  - `PUT /api/boards/{boardId}/permissions/{userId}` — Change role.
  - `DELETE /api/boards/{boardId}/permissions/{userId}` — Remove collaborator.

**Enums**

- `BoardRole` — owner, editor, viewer.
- `BoardVisibility` — private, shared, public.

------

### **2.4 Sections & Tabs**

| Component          | Purpose                                             |
| ------------------ | --------------------------------------------------- |
| **SectionService** | Manages board sections (columns, groups).           |
| **TabService**     | Manages per-section or board tabs (views, layouts). |

**Repositories**

- `SectionRepository`
- `TabRepository`

**Controllers**

- `SectionsController`
  - `GET /api/boards/{boardId}/sections` — List sections.
  - `POST /api/boards/{boardId}/sections` — Create section.
  - `PUT /api/boards/{boardId}/sections/{id}` — Update title or position.
  - `DELETE /api/boards/{boardId}/sections/{id}` — Delete section.
- `TabsController`
  - `GET /api/boards/{boardId}/tabs` — List tabs.
  - `POST /api/boards/{boardId}/tabs` — Create new tab.
  - `PUT /api/boards/{boardId}/tabs/{id}` — Update title or layout.
  - `DELETE /api/boards/{boardId}/tabs/{id}` — Delete tab.

**Enums**

- `TabScope` — board, section.

------

### **2.5 Cards & Relations**

| Component                | Purpose                                       |
| ------------------------ | --------------------------------------------- |
| **CardService**          | CRUD, search, filtering, assignment.          |
| **CardRelationsService** | Links, tags, and relationships between cards. |

**Repositories**

- `CardRepository`
- `TagRepository`
- `CardTagRepository`
- `LinkRepository`

**Controllers**

- `CardsController`
  - `GET /api/boards/{boardId}/cards` — List all cards for board.
  - `GET /api/boards/{boardId}/sections/{sectionId}/cards` — Cards per section.
  - `POST /api/boards/{boardId}/cards` — Create card.
  - `PUT /api/boards/{boardId}/cards/{id}` — Update card fields.
  - `DELETE /api/boards/{boardId}/cards/{id}` — Delete card.
  - `GET /api/boards/{boardId}/cards/search?keyword=` — Search cards.
- `CardRelationsController`
  - `POST /api/cards/{id}/tags` — Add tag.
  - `DELETE /api/cards/{id}/tags/{tagId}` — Remove tag.
  - `POST /api/cards/{id}/links` — Link cards.
  - `DELETE /api/cards/{id}/links/{linkId}` — Remove link.

**Enums**

- `CardType` — note, task, event, reminder.
- `CardStatus` — todo, in_progress, done.
- `LinkType` — reference, dependency, mirror.

------

### **2.6 Organizations & Members**

| Component               | Purpose                                                 |
| ----------------------- | ------------------------------------------------------- |
| **OrganizationService** | Unified handler for organization and member management. |

**Repositories**

- `OrganizationRepository`
- `OrganizationMemberRepository`

**Controllers**

- `OrganizationsController`
  - `GET /api/organizations` — List user organizations.
  - `POST /api/organizations` — Create organization.
  - `PUT /api/organizations/{id}` — Update metadata.
  - `DELETE /api/organizations/{id}` — Delete organization.
  - `GET /api/organizations/{id}/members` — List members.
  - `POST /api/organizations/{id}/members` — Invite or add member.
  - `PUT /api/organizations/{id}/members/{userId}` — Update member role.
  - `DELETE /api/organizations/{id}/members/{userId}` — Remove member.

**Enums**

- `OrgRole` — owner, admin, member.

------

### **2.7 Automation: Rules & Clusters**

| Component          | Purpose                                                      |
| ------------------ | ------------------------------------------------------------ |
| **RuleService**    | Defines automation logic as JSON definitions tied to a board. |
| **ClusterService** | Groups or auto-categorizes cards based on `ClusterType` and optional rules. |

**Repositories**

- `RuleRepository`
- `ClusterRepository`

**Controllers**

- `RulesController`
  - `GET /api/boards/{boardId}/rules`
  - `POST /api/boards/{boardId}/rules`
  - `PUT /api/boards/{boardId}/rules/{id}`
  - `DELETE /api/boards/{boardId}/rules/{id}`
- `ClustersController`
  - `GET /api/boards/{boardId}/clusters`
  - `POST /api/boards/{boardId}/clusters`
  - `PUT /api/boards/{boardId}/clusters/{id}`
  - `DELETE /api/boards/{boardId}/clusters/{id}`

**Enums**

- `ClusterType` — manual, rule_based, ai_generated.

------

### **2.8 Activities (Audit Trail)**

| Component           | Purpose                                  |
| ------------------- | ---------------------------------------- |
| **ActivityService** | Append-only event log per board or card. |

**Repository**

- `ActivityRepository`

**Controller**

- `ActivitiesController`
  - `GET /api/boards/{boardId}/activities` — Get all activities for a board.
  - `GET /api/boards/{boardId}/activities/cards/{cardId}` — Get activities for a card.
  - `POST /api/boards/{boardId}/activities` — Log an activity (internal use).
  - `GET /api/activities/recent?limit=20` — Admin dashboard feed.
  - `DELETE /api/boards/{boardId}/activities/{id}` — Admin cleanup.

**Enums**

- `ActivityType` — card_created, card_updated, section_moved, comment_added, etc.

------

## **3. Infrastructure**

### **3.1 Dependency Injection**

- `NpgsqlDataSource` → Singleton (shared connection pool)
- Repositories → Scoped
- Services → Scoped
- Controllers → Scoped (default)

### **3.2 Enum Mapping**

All enums registered in `Program.cs` using `dataSourceBuilder.MapEnum<T>()`.

### **3.3 Authentication**

- JWT bearer with issuer/audience/secret from configuration.
- `Authorize` attributes enforce endpoint security.
- `AdminOnly` policy for privileged routes.

### **3.4 Swagger**

- Fully integrated with bearer token authentication.
- Cache-busting enabled in development for version updates.
- Available at `/api/swagger`.

------

## **4. Status Summary**

| Category              | Completion | Notes                                               |
| --------------------- | ---------- | --------------------------------------------------- |
| Authentication        | ✅          | Working with refresh tokens                         |
| Users                 | ✅          | CRUD and search                                     |
| Messaging & Relations | ✅          | Full collaboration pipeline                         |
| Boards & Permissions  | ✅          | Role and visibility enforced                        |
| Sections & Tabs       | ✅          | Positioning and layout metadata                     |
| Cards, Tags & Links   | ✅          | Rich content and relations                          |
| Organizations         | ✅          | Merged with members                                 |
| Rules & Clusters      | ✅          | Automation and grouping                             |
| Activities            | ✅          | Append-only audit system                            |
| Worker / Jobs         | ⏳ Pending  | Queuing and background tasks to be implemented next |