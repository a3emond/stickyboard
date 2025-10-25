# StickyBoard – Project Overview (2025-10-24)

StickyBoard is a **cross-platform collaborative workspace** designed for creativity, productivity, and intelligent organization. It unifies personal notes, project boards, and team collaboration into a single local-first ecosystem with seamless offline and cloud sync.

This repository acts as the **documentation and coordination hub** for the entire StickyBoard ecosystem, providing architecture references and linking platform-specific submodules.

| Repository                                                   | Platform | Description                                     |
| ------------------------------------------------------------ | -------- | ----------------------------------------------- |
| [stickyboard-api](https://github.com/a3emond/stickyboard-api) | Backend  | ASP.NET Core REST API + PostgreSQL (Dockerized) |
| [stickyboard-ios](https://github.com/a3emond/stickyboard-ios) | iOS      | Native SwiftUI app for iPhone and iPad          |
| [stickyboard-android](https://github.com/a3emond/stickyboard-android) | Android  | Native Kotlin app                               |
| [stickyboard-macos](https://github.com/a3emond/stickyboard-macos) | macOS    | Desktop version built in SwiftUI                |
| [stickyboard-windows](https://github.com/a3emond/stickyboard-windows) | Windows  | Desktop version built in .NET MAUI              |

------

## 1. Architecture Overview

### 1.1 Core Principles

- **Local-first** — instant response, offline by default, sync when connected.
- **Composable domain** — boards, sections, cards, and tabs behave uniformly.
- **Operation-based sync** — every mutation is logged as a reversible operation.
- **Extensible schema** — new entities can be introduced without breaking existing data.
- **Collaborative by design** — same structure powers personal and team workflows.

### 1.2 Major Layers

1. **Data Storage** – Local database + remote persistence (PostgreSQL).
2. **Sync Layer** – Operation logs with versioned merges (CRDT-inspired).
3. **Domain Layer** – Core objects (Board, Section, Card, Cluster, User, etc.).
4. **Rule Engine** – Local automation and inference.
5. **Collaboration Layer** – Boards, organizations, and messaging.
6. **Presentation Layer** – Unified visual canvas and cross-platform UI.

------

## 2. Collaboration Model

### 2.1 Structure

StickyBoard introduces a **multi-tier collaboration system** combining organizations, teams, and personal spaces:

```
User
 ├── Friends (Relations)
 ├── Messages (Invites, Notifications)
 ├── Organizations
 │    ├── Members (OrgRole)
 │    └── Boards (Projects)
 │          ├── Shared Board (Public project)
 │          └── Personal Boards (Private per member)
 ├── Boards (Personal)
 │    └── Permissions (BoardRole)
 └── Invites (External email entrypoints)
```

### 2.2 Roles

**Organization Roles**

| Role      | Access                          |
| --------- | ------------------------------- |
| Owner     | Full control, cannot be removed |
| Admin     | Manage org and boards           |
| Moderator | Manage members and permissions  |
| Member    | Standard participant            |
| Guest     | Read-only                       |

**Board Roles**

| Role      | Access                       |
| --------- | ---------------------------- |
| Owner     | Full control                 |
| Editor    | CRUD permissions, can invite |
| Commenter | Comment and view             |
| Viewer    | Read-only                    |

### 2.3 Boards as Projects

Boards can act as **project containers** with a shared parent board and personal subboards per user. Owners have read-only access to member subboards, while admins can view all. This structure supports both individual productivity and coordinated teamwork.

### 2.4 Invitations and Messaging

- **Internal Invites** – for existing users; instantly creates permissions.
- **External Invites** – generates a secure token + email with download links.
- **Messaging System** – handles invites, direct messages, and system notifications.
- **Friend Relations** – allow user discovery and restricted invites.

### 2.5 Offline and Sync

All collaboration actions (invites, joins, permission changes) produce `operations` and `activities`, ensuring deterministic sync and offline resilience.

------

## 3. Data Model Summary

| Entity                 | Description                                              |
| ---------------------- | -------------------------------------------------------- |
| **User**               | Identity, profile, preferences                           |
| **Organization**       | Team workspace container                                 |
| **OrganizationMember** | Relation between User and Organization (role, joined_at) |
| **Board**              | Workspace/project; may belong to org or user             |
| **Permission**         | Board-level collaborator roles                           |
| **Section**            | Visual grouping of cards within board                    |
| **Tab**                | Contextual sub-area of board or section                  |
| **Card**               | Atomic note/task/event element                           |
| **Cluster**            | Logical grouping (manual or rule-based)                  |
| **Activity**           | Log of domain actions                                    |
| **Rule**               | Declarative automation definition                        |
| **Message**            | Communication item (invite, DM, system)                  |
| **Invite**             | External invitation link                                 |
| **UserRelation**       | Friend or blocked connection between users               |
| **WorkerJob**          | Asynchronous task in background queue                    |

------

## 4. Collaboration Features Overview

- **Organizations** – Manage multi-user workspaces with role hierarchy.
- **Board Sharing** – Invite others with roles; integrate into org access.
- **Personal Boards** – Each user can have private subboards under shared projects.
- **Friend System** – Basic social layer for invitations and presence.
- **Messaging** – Central hub for all communication, notifications, and invites.
- **External Invites** – Email-based onboarding for non-members.
- **Audit Trail** – Every collaboration action logged as an `Activity`.
- **Offline Support** – Full local caching and deferred operation queue.

------

## 5. Sync Lifecycle

1. Local action → create operation entry.
2. Operation persisted to local DB.
3. Optional rule triggers → enqueue background jobs.
4. Background sync pushes operations.
5. API merges → emits events for subscribed clients.
6. UI layer refreshes via observable model streams.

Undo/redo handled locally through inverse operations.

------

## 6. Intelligence & Rule Layer

Local subsystems enhance notes and boards with automation:

| Subsystem         | Input              | Output                           |
| ----------------- | ------------------ | -------------------------------- |
| Rule Engine       | Card/Board updates | Triggers, suggestions, auto-tags |
| Ink Parser        | Drawn content      | Vector + recognized text         |
| Similarity Engine | Text metadata      | Related cards and clusters       |
| Index Builder     | Text + tags        | Full-text search index           |
| Cluster Manager   | Rule + embeddings  | Grouped card sets                |

------

## 7. Product Vision

StickyBoard aims to become the **most flexible hybrid between a visual note system and a team collaboration platform**:

- Personal creativity meets organizational coordination.
- Every device stays productive offline.
- Transparent sync, open JSON data, and exportable archives.
- Modular backend with PostgreSQL + Worker Jobs.
- Multi-platform clients sharing one universal model.

### Roadmap Phases

| Phase | Goal                 | Deliverables                                    |
| ----- | -------------------- | ----------------------------------------------- |
| 1     | Personal MVP         | Board CRUD, offline DB, operation log           |
| 2     | Intelligence         | Rules, handwriting recognition, search clusters |
| 3     | Collaboration        | Boards, organizations, invites, messages        |
| 4     | Multi-Board Projects | Parent boards, personal subboards               |
| 5     | Ecosystem            | Plugins, integrations, analytics workers        |

------

## 8. Governance & Data Portability

- Open JSON schema with versioning.
- Portable `.stickyboard` archives (data + assets).
- Optional end-to-end encryption for private boards.
- Auditable operation history via `operations` + `activities`.
- Export adapters for Markdown, CSV, and iCalendar.

------

## License

All rights reserved — © Alexandre Emond, 2025. Unauthorized reproduction or redistribution of this project’s source code, in whole or in part, is strictly prohibited without express written permission.
