# StickyBoard – Project Overview

StickyBoard is a next-generation, cross-platform visual organization and note management system designed for creativity, knowledge mapping, and collaborative thinking. It combines local-first reliability, intelligent automation, and multi-device synchronization to create a seamless workspace experience.

This repository acts as the **main documentation and coordination hub** for the StickyBoard ecosystem, linking together multiple platform-specific submodules:

| Repository                                                            | Platform | Description                                     |
| --------------------------------------------------------------------- | -------- | ----------------------------------------------- |
| [stickyboard-api](https://github.com/a3emond/stickyboard-api)         | Backend  | ASP.NET Core REST API + PostgreSQL (Dockerized) |
| [stickyboard-ios](https://github.com/a3emond/stickyboard-ios)         | iOS      | Native SwiftUI app for iPhone and iPad          |
| [stickyboard-android](https://github.com/a3emond/stickyboard-android) | Android  | Native Java                                     |
| [stickyboard-macos](https://github.com/a3emond/stickyboard-macos)     | macOS    | Desktop version built in SwiftUI                |
| [stickyboard-windows](https://github.com/a3emond/stickyboard-windows) | Windows  | Desktop version                                 |

---

## 1. Overall Architecture Model

### 1.1 Core Principles

* **Local-first**: every action persists locally, syncs opportunistically.
* **Resource-oriented**: all entities addressable as independent resources (`/boards/:id`, `/cards/:id`).
* **Composable features**: cards, sections, and clusters behave the same across devices.
* **Reactive UI model**: any state change (local or remote) propagates through a single data stream.
* **Extensible data contracts**: new fields and types can be added without schema breaks.

### 1.2 Major Layers

1. **Storage Layer** – local database + remote persistence adapters.
2. **Sync & Versioning Layer** – CRDT/operation log, delta-based sync, conflict merge.
3. **Domain Layer** – core models: Board, Section, Card, Cluster, User, etc.
4. **Rule & Intelligence Layer** – parsers, rule engine, search/index, handwriting recognizer.
5. **Collaboration Layer** – permissions, live presence, comments, activities.
6. **Presentation Layer** – board canvas, list views, filters/lenses, search UI.
7. **API Gateway Layer** – REST endpoints and real-time streams (for later multi-device sync).

---

## 2. Data Model Precision

### 2.1 Core Entities (normalized)

| Entity           | Core Fields                                                                                          | Relationships                                                             |
| ---------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **User**         | id, name, email, avatarUri, prefs                                                                    | owns Boards; participates via Collaborator                                |
| **Board**        | id, title, visibility, createdBy, tabs[], rules[], theme                                             | contains Sections, Cards, Collaborators                                   |
| **Section**      | id, title, position, layoutMeta                                                                      | belongs to Board; holds Cards                                             |
| **Tab**          | id, title, type (“plan”, “archive”), layoutConfig                                                    | belongs to Board/Section                                                  |
| **Card**         | id, type, title, content, inkData, date, tags[], assignees[], status, priority, createdAt, updatedAt | belongs to Section or Tab; linked via LinkGraph; referenced by Activities |
| **Cluster**      | id, ruleDefinition, type (auto/manual), members[], visualMeta                                        | references Cards; belongs to Board                                        |
| **Collaborator** | userId, boardId, role                                                                                | —                                                                         |
| **Activity**     | id, boardId, cardId?, type (edit/comment/react), actor, payload, timestamp                           | —                                                                         |
| **Rule**         | id, scope (board/section/global), trigger, condition, action                                         | executed by local Rule Engine                                             |
| **Link**         | id, fromCard, toCard, type (reference, duplicate, depends)                                           | drives knowledge graph                                                    |

---

## 3. Data Flow and State Lifecycle

1. User action (create/edit/move) → generates a domain operation (`AddCard`, `MoveCard`, `EditField`, etc.).
2. Operation logged locally (append-only).
3. Rule Engine runs pre-commit checks (auto-tag, link inference).  \
4. Operation applied to local DB (Room/SQLite/IndexedDB).
5. Background Sync Service batches logs → remote server.
6. Server merges ops, broadcasts deltas back to subscribers.
7. UI reacts through observable data streams.

Undo/redo: revert via inverse operations from the local log.

---

## 4. Local Intelligence Subsystems

| Subsystem              | Input                         | Output                                      | Notes                               |
| ---------------------- | ----------------------------- | ------------------------------------------- | ----------------------------------- |
| Ink Processor          | pen strokes                   | raster preview + vector data                | stores compressed path data         |
| Handwriting Recognizer | inkData                       | recognizedText                              | local ML model or OS API            |
| Parser                 | raw text                      | structured metadata (dates, tags, mentions) | rule-based regex + tokenization     |
| Rule Engine            | card state changes            | new tags, reminders, links, nudges          | event-driven, declarative DSL       |
| Index Builder          | card text, metadata           | FTS index + inverted word map               | incremental background job          |
| Similarity Engine      | text vectors                  | related-cards suggestions                   | TF-IDF or local embeddings          |
| Cluster Manager        | similarity index + user rules | cluster membership sets                     | dynamic; persists cluster snapshots |

Each subsystem communicates through an Event Bus: `onCardUpdated`, `onCardCreated`, `onIndexRebuilt`, `onRuleTriggered`, etc.

---

## 5. Rule Engine Model

**Rule Definition:**

```
{
  trigger: "onCardUpdate",
  condition: { field: "content", contains: "meeting" },
  actions: [
    { type: "addTag", value: "work" },
    { type: "suggestConvert", targetType: "event" }
  ],
  scope: "board:123"
}
```

Execution model:

1. Observe changes.
2. Evaluate condition.
3. Queue or apply actions (depending on “auto” vs “suggest”).

User can enable/disable per rule; engine remains deterministic and offline.

---

## 6. Search & Indexing Architecture

### 6.1 Index Types

* FTS index: token → cardId list.
* Metadata index: tag/date/type/assignee → cardId.
* Vector index (optional): TF-IDF or embedding array → cosine similarity.

### 6.2 Search Flow

1. Parse query → filter tokens, metadata, operators.
2. Merge hits from all indexes.
3. Rank by recency, similarity, and user context (board, section).
4. Present grouped results (by board/section/type).

### 6.3 Incremental updates

* On every card change, update only affected tokens and metadata.
* Low-priority background thread handles re-indexing.

---

## 7. Collaboration Model (Offline-Tolerant)

### 7.1 Roles

* Owner → full control.
* Editor → CRUD within scope.
* Commenter → read + comment.
* Viewer → read only.

### 7.2 Conflict Resolution

* Content fields → text diff + merge preview.
* Position changes → CRDT sequence positions.
* Deletions → tombstones with version stamps.

### 7.3 Presence Channels

* Lightweight real-time events: cursor positions, selections, comments.
* Optional; falls back to polling or no-presence when offline.

---

## 8. Security & Privacy Layer

* Local encryption: per-board key derived from user keychain.
* Selective end-to-end: cards/sections flagged as private encrypt at client before sync.
* Permissions enforcement: at domain layer before any API call.
* Audit trail: immutable append log of activities, exportable as JSON/CSV.

---

## 9. Frontend Functional Architecture

### 9.1 Components

* Board Canvas: zoomable, pannable layout; houses sections and cards.
* Sidebar Panels: Filters, Search, Activity, Cluster view.
* Card Editor: inline modal; supports text + ink + attachments.
* Timeline View: alternative layout focusing on dated cards.
* Quick Capture Overlay: floating bubble accessible globally.

### 9.2 Interaction Model

* Pen/touch gesture → create note.
* Drag → move card/section.
* Tap/hold → open context menu (convert type, link, duplicate).
* Keyboard shortcuts → filter, create, switch view.
* Search bar → universal command palette (`/create task`, `#tag`, `@assign`).

---

## 10. Sync & Version Semantics

* Operation log: `[opId, entityId, type, payload, timestamp, userId]`.
* Version vectors: per-entity version for merging.
* Delta endpoint: `GET /sync?since=timestamp` returns operations list.
* Merge rules: LWW per field; CRDT per position; additive for comments/links.

Local client keeps `lastSync` cursor per board; retry policy exponential with jitter.

---

## 11. Extensibility Points

* Plugin hooks: allow local add-ons (e.g., custom rule packs or renderers).
* Custom card types: define schema extensions and rendering functions.
* Integration hooks: generic “sync adapters” (Calendar, Email, etc.) that operate through uniform interface.

All run in sandboxed context so they can be added without refactoring core.

---

## 12. Product & UX Roadmap

| Phase                      | Focus                                                        | Key Deliverables        |
| -------------------------- | ------------------------------------------------------------ | ----------------------- |
| 1. Personal MVP            | Core CRUD, board canvas, offline persistence, keyword search | stable base             |
| 2. Local Intelligence      | handwriting → text, rule engine, clustering, related view    | smart offline assistant |
| 3. Collaboration Alpha     | sharing, comments, live cursors, basic permission checks     | team use                |
| 4. Search & Rule Expansion | FTS + vector index, saved lenses, automation UI              | power-user stage        |
| 5. Multi-Board Projects    | board clusters, rollups, project summaries                   | scaling stage           |
| 6. Ecosystem               | plugin system, cloud AI extensions, enterprise controls      | maturity                |

---

## 13. Governance & Data Portability

* Data format: open JSON spec with version field.
* Export/Import: single-file `.stickyboard` archive (JSON + assets).
* Versioning: schema versioning via migration descriptors.
* Interoperability: optional adapter to/from Markdown, CSV, or iCalendar.


---

## License

All rights reserved — © Alexandre Emond, 2025. Unauthorized copying or redistribution of this project’s source code, in whole or in part, is strictly prohibited without express written permission.
