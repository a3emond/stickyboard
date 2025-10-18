# StickyBoard — Client Application Architecture & Data Flow Definition

*Last updated: 2025-10-17*

This document defines the **client-side architecture** for all StickyBoard consuming applications (Android tablet, web, desktop, mobile companion). It describes the structure, data flow, offline synchronization model, local storage, and interaction patterns with the backend REST API.

The design is **platform-agnostic** and can be implemented in Kotlin (Android), Swift (iOS/macOS), JavaScript/TypeScript (Web), or cross-platform frameworks (Flutter, Compose Multiplatform, .NET MAUI, etc.).

------

## 1) Architectural Overview

The client is built around a **local-first, reactive architecture**:

```
UI Layer (Views, Components)
│
├─ ViewModel / Controller Layer
│   ├─ State management & reactive updates
│   └─ Calls UseCases and DataRepository interfaces
│
├─ Domain Layer (UseCases, Business Rules)
│
└─ Data Layer
     ├─ Local Store (SQLite + Cache)
     ├─ Remote API Client (REST)
     ├─ Sync Engine
     └─ File/Attachment Store
```

### Key Goals

- Offline-first usability.
- Unified data model across devices.
- Reactive, real-time updates when sync completes.
- Minimal API calls with batching & caching.
- Separation of concerns for clean maintainability.

------

## 2) Core Modules

### 2.1 UI Layer

- Platform-specific presentation.
- Responsive design for tablets (primary) and mobile/desktop variants.
- Key Components:
  - **BoardCanvasView** — Freeform workspace for sections, tabs, and cards.
  - **SectionListView** — List-based view for sections and contained cards.
  - **CardEditorView** — Rich editor for note/task/event content + pen input.
  - **TaskListView** — Aggregated view of all tasks by due date.
  - **SearchView** — Unified search and filters.
  - **SettingsView** — Preferences, themes, and sync configuration.

### 2.2 ViewModel / Controller Layer

Implements **reactive state management**.

- Observes LiveData/Flows/Signals from Repositories.
- Handles UI events and converts them into actions or operations.
- Maintains ephemeral UI state (filters, selections, unsaved edits).

**Examples:**

- `BoardViewModel` → loads board, sections, tabs.
- `CardViewModel` → manages card CRUD, tagging, and ink recognition.
- `SyncViewModel` → monitors sync progress and conflict resolution.

### 2.3 Domain Layer (UseCases)

Encapsulates core app logic, independent of UI and persistence.

| UseCase              | Description                                      |
| -------------------- | ------------------------------------------------ |
| `CreateCardUseCase`  | Create a card locally, enqueue for sync.         |
| `UpdateCardUseCase`  | Apply patch, version increment.                  |
| `MoveCardUseCase`    | Change card section/tab, maintain operation log. |
| `SearchCardsUseCase` | Execute local FTS query; fallback to remote.     |
| `ApplyRuleUseCase`   | Evaluate local rules post-update.                |
| `SyncBoardUseCase`   | Pull/push delta operations for a board.          |
| `ExportBoardUseCase` | Generate .stickyboard archive.                   |

UseCases interact with repositories only.

### 2.4 Data Layer

Implements persistence and API communication.

#### 2.4.1 Local Store

- SQLite via ORM/DAO layer (Room, CoreData, etc.).
- Mirrors the server schema for `boards`, `sections`, `tabs`, `cards`, etc.
- Maintains an **operations** table for unsynced changes.
- Includes an **FTS index** for fast offline search.

#### 2.4.2 Remote API Client

- HTTP client for all `/api/...` endpoints.
- Handles authentication tokens (JWT) and retries.
- Supports batch operations for sync.
- Error parsing and normalization (`ERR_*` codes → user-friendly messages).

#### 2.4.3 Sync Engine

Manages delta-based bidirectional synchronization.

- Pulls operations (`GET /api/sync?since`)
- Pushes local operations (`POST /api/sync`)
- Detects conflicts via `versionPrev` mismatch.
- Applies merges and updates local DB.
- Queues retries when offline.

**Sync Strategies:**

1. **Passive**: auto-sync every X minutes.
2. **Active**: user-triggered pull/push.
3. **Realtime** (optional): WebSocket event stream updates.

#### 2.4.4 File Manager

- Handles uploads/downloads of attachments.
- Local caching of thumbnails.
- Retry queue for failed uploads.

------

## 3) Data Flow

### 3.1 Local CRUD Flow

```
User edits card → ViewModel → UseCase → Repository (Local) →
Operation logged → UI updates immediately →
Sync Engine pushes when online → Backend persists → Version incremented.
```

### 3.2 Sync Flow

```
[Device]
  ↓ Pull / Push via SyncService
[API Server]
  ↓ Validate / Merge / Broadcast
[Other Devices]
  ↓ Receive delta → Apply locally → Refresh UI
```

### 3.3 Conflict Handling

- Field-level comparison.
- UI highlights conflicting fields.
- User chooses “Keep mine” or “Accept server.”
- Merge result queued as a new operation.

### 3.4 Rule Execution

- After any local card update, `ApplyRuleUseCase` runs synchronously.
- Matching rules trigger auto-tags or reminders.
- Suggested actions displayed non-intrusively in UI.

------

## 4) Offline Behavior

| Scenario                | Expected Behavior                                           |
| ----------------------- | ----------------------------------------------------------- |
| **No internet**         | All CRUD ops stored in local DB and queued in `operations`. |
| **Reconnect**           | SyncEngine flushes queue and pulls deltas.                  |
| **Conflict**            | Version mismatch → flagged for resolution.                  |
| **Token expired**       | Stored refresh token used for renewal.                      |
| **Attachments offline** | Thumbnails visible; upload deferred.                        |

------

## 5) Module Boundaries

| Module            | Responsibilities                            |
| ----------------- | ------------------------------------------- |
| `core-models`     | Shared DTOs, enums, JSON serializers.       |
| `data-repository` | Local + Remote accessors, sync logic.       |
| `domain-usecases` | Business operations and rules.              |
| `ui`              | View components and interactions.           |
| `intelligence`    | Local handwriting recognition, NLP parsing. |
| `sync`            | Background worker, conflict resolver.       |

------

## 6) Local Intelligence Layer (Client-Side)

Local, offline-first logic mirroring server-side Intelligence.

- Handwriting recognition via platform API (Android ML Kit, iOS VisionKit, etc.).
- Natural language parsing for quick due dates, tags, and assignees.
- Local rule execution.
- TF-IDF search index maintenance.

**Key Components:**

- `InkProcessor`
- `TextParser`
- `RuleEvaluator`
- `SimilarityEngine`
- `Indexer`

All operate on local DB changes and emit events for the UI.

------

## 7) Security & Privacy (Client-Side)

- Token stored in secure storage (Keychain/Keystore).
- Private cards encrypted locally before sync.
- Export archives (.stickyboard) password-encrypted.
- Logs sanitized before sharing.

------

## 8) Background Services

| Service            | Function                                          |
| ------------------ | ------------------------------------------------- |
| `SyncWorker`       | Periodic push/pull sync with exponential backoff. |
| `IndexWorker`      | Maintains FTS index from recent changes.          |
| `RuleWorker`       | Executes scheduled or delayed rules.              |
| `BackupWorker`     | Creates local backup snapshot daily.              |
| `FileUploadWorker` | Handles deferred file uploads.                    |

------

## 9) Communication Layer

**HTTP API:** Standard JSON REST requests to backend endpoints.
 **WebSocket/SSE:** Optional live updates for shared boards.
 **Notifications:** Platform-specific push notifications for mentions and due tasks.

------

## 10) Cross-Platform Considerations

| Platform                | Key Notes                                                |
| ----------------------- | -------------------------------------------------------- |
| Android                 | Jetpack Compose UI; Room DB; WorkManager for background. |
| iOS/macOS               | SwiftUI + CoreData; background sync via URLSessionTasks. |
| Web                     | IndexedDB + Service Workers; PWA mode for offline use.   |
| Desktop (Electron/MAUI) | SQLite local store; background sync thread.              |

------

## 11) Development Guidelines

- Keep all DTOs identical to backend canonical JSON.
- Always perform optimistic UI updates.
- Store last sync timestamp per board.
- Log every operation (create/update/delete/move) with UUID.
- Modularize the app for testability (mockable repositories and API clients).

------

## 12) Future Extensions

- Multi-user collaboration presence (real-time cursors).
- End-to-end encrypted shared boards.
- AI-assisted local summaries (optional add-on).
- Plugin system for custom UI components.

------

### End of Client Application Architecture Document