# StickyBoard — Data & Storage Specification (PostgreSQL + SQLite + Canonical JSON)

*Last updated: 2025‑10‑17*

This document defines the **data model**, **database schemas** (PostgreSQL server + SQLite local), **canonical JSON contracts**, **indexing/FTS**, and **sync/versioning** semantics for StickyBoard. It is **stack-agnostic** and optimized for a **local‑first, offline‑capable** architecture.

------

## 0) Design Principles

- **Relational core** for integrity (users, boards, sections, permissions, activities).
- **JSON payloads** (content, layout, rules) for flexible evolution.
- **Local‑first** mirroring: SQLite mirrors Postgres schema as closely as practical.
- **Immutable operation log** to drive sync, undo/redo, and audit.
- **Searchable** via FTS (Postgres `tsvector`, SQLite `FTS5`).
- **Portable** via a single **canonical JSON** source of truth.

------

## 1) Entity Overview

**Primary entities**

- `users`
- `boards`
- `sections`
- `tabs`
- `cards`
- `links`
- `clusters`
- `permissions`
- `activities`
- `rules`
- `files` (attachments)
- `operations` (sync log)

**Derived / system artifacts**

- FTS materializations (server + local)
- Materialized views for rollups / cluster membership

------

## 2) PostgreSQL Schema

> Types used: `uuid`, `timestamptz`, `jsonb`, `text`, `integer`, `boolean`. Arrays used sparingly; prefer join tables for portability.

### 2.1 Users

```sql
create table users (
  id             uuid primary key,
  email          text not null unique,
  display_name   text not null,
  avatar_uri     text,
  prefs          jsonb not null default '{}'::jsonb,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now()
);
create index idx_users_email_trgm on users using gin (email gin_trgm_ops);
create trigger trg_users_updated_at before update on users
  for each row execute procedure set_updated_at();
```

### 2.2 Boards

```sql
create type board_visibility as enum ('private','shared','public');

create table boards (
  id            uuid primary key,
  owner_id      uuid not null references users(id) on delete cascade,
  title         text not null,
  visibility    board_visibility not null default 'private',
  theme         jsonb not null default '{}'::jsonb,
  rules         jsonb not null default '[]'::jsonb,
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now()
);
create index idx_boards_owner on boards(owner_id);
create trigger trg_boards_updated_at before update on boards
  for each row execute procedure set_updated_at();
```

### 2.3 Permissions (Collaborators)

```sql
create type board_role as enum ('owner','editor','commenter','viewer');

create table permissions (
  user_id   uuid not null references users(id) on delete cascade,
  board_id  uuid not null references boards(id) on delete cascade,
  role      board_role not null,
  granted_at timestamptz not null default now(),
  primary key (user_id, board_id)
);
create index idx_permissions_board on permissions(board_id);
```

### 2.4 Sections

```sql
create table sections (
  id           uuid primary key,
  board_id     uuid not null references boards(id) on delete cascade,
  title        text not null,
  position     integer not null,
  layout_meta  jsonb not null default '{}'::jsonb,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);
create index idx_sections_board_pos on sections(board_id, position);
create trigger trg_sections_updated_at before update on sections
  for each row execute procedure set_updated_at();
```

### 2.5 Tabs (Board‑level or Section‑bound)

```sql
create type tab_scope as enum ('board','section');

create table tabs (
  id            uuid primary key,
  scope         tab_scope not null,
  board_id      uuid not null references boards(id) on delete cascade,
  section_id    uuid references sections(id) on delete set null,
  title         text not null,
  tab_type      text not null default 'custom',
  layout_config jsonb not null default '{}'::jsonb,
  position      integer not null,
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now(),
  constraint tabs_scope_consistency check ((scope='board' and section_id is null) or (scope='section' and section_id is not null))
);
create index idx_tabs_board_pos on tabs(board_id, position);
create trigger trg_tabs_updated_at before update on tabs
  for each row execute procedure set_updated_at();
```

### 2.6 Cards

```sql
create type card_type as enum ('note','task','event','drawing');
create type card_status as enum ('open','in_progress','blocked','done','archived');

create table cards (
  id           uuid primary key,
  board_id     uuid not null references boards(id) on delete cascade,
  section_id   uuid references sections(id) on delete set null,
  tab_id       uuid references tabs(id) on delete set null,
  type         card_type not null,
  title        text,
  content      jsonb not null default '{}'::jsonb,   -- rich text, blocks, checklists, etc.
  ink_data     jsonb,                                -- vector strokes + snapshots
  due_date     timestamptz,
  start_time   timestamptz,
  end_time     timestamptz,
  priority     integer,
  status       card_status not null default 'open',
  created_by   uuid not null references users(id) on delete set null,
  assignee_id  uuid references users(id) on delete set null,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now(),
  version      integer not null default 0
);
create index idx_cards_board on cards(board_id);
create index idx_cards_section on cards(section_id);
create index idx_cards_due on cards(due_date);
create index idx_cards_status on cards(status);
create trigger trg_cards_updated_at before update on cards
  for each row execute procedure set_updated_at();
```

#### 2.6.a Card Tags (normalized)

```sql
create table tags (
  id    uuid primary key,
  name  text not null unique
);

create table card_tags (
  card_id uuid not null references cards(id) on delete cascade,
  tag_id  uuid not null references tags(id) on delete cascade,
  primary key (card_id, tag_id)
);
create index idx_card_tags_tag on card_tags(tag_id);
```

#### 2.6.b Card Assignees (multi‑assignee)

```sql
create table card_assignees (
  card_id uuid not null references cards(id) on delete cascade,
  user_id uuid not null references users(id) on delete cascade,
  primary key (card_id, user_id)
);
```

### 2.7 Links (Knowledge Graph)

```sql
create type link_type as enum ('references','duplicate_of','relates_to','blocks','depends_on');

create table links (
  id         uuid primary key,
  from_card  uuid not null references cards(id) on delete cascade,
  to_card    uuid not null references cards(id) on delete cascade,
  rel_type   link_type not null,
  created_at timestamptz not null default now(),
  created_by uuid references users(id) on delete set null
);
create index idx_links_from on links(from_card);
create index idx_links_to on links(to_card);
```

### 2.8 Clusters

```sql
create type cluster_type as enum ('manual','rule','similarity');

create table clusters (
  id           uuid primary key,
  board_id     uuid not null references boards(id) on delete cascade,
  cluster_type cluster_type not null,
  rule_def     jsonb,                 -- present if type='rule'
  visual_meta  jsonb not null default '{}'::jsonb,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

create table cluster_members (
  cluster_id uuid not null references clusters(id) on delete cascade,
  card_id    uuid not null references cards(id) on delete cascade,
  primary key (cluster_id, card_id)
);
create index idx_cluster_members_card on cluster_members(card_id);
```

### 2.9 Activities

```sql
create type activity_type as enum (
  'card_created','card_updated','card_moved','comment_added','status_changed','assignee_changed','link_added','link_removed','rule_triggered'
);

create table activities (
  id         uuid primary key,
  board_id   uuid not null references boards(id) on delete cascade,
  card_id    uuid references cards(id) on delete set null,
  actor_id   uuid references users(id) on delete set null,
  act_type   activity_type not null,
  payload    jsonb not null default '{}'::jsonb,
  created_at timestamptz not null default now()
);
create index idx_activities_board_time on activities(board_id, created_at desc);
```

### 2.10 Rules

```sql
create table rules (
  id         uuid primary key,
  board_id   uuid not null references boards(id) on delete cascade,
  definition jsonb not null,
  enabled    boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create index idx_rules_board on rules(board_id);
```

### 2.11 Files (Attachments)

```sql
create table files (
  id          uuid primary key,
  owner_id    uuid not null references users(id) on delete cascade,
  board_id    uuid references boards(id) on delete set null,
  card_id     uuid references cards(id) on delete set null,
  storage_key text not null,     -- object key in S3/minio
  filename    text not null,
  mime_type   text,
  size_bytes  bigint,
  meta        jsonb not null default '{}'::jsonb,
  created_at  timestamptz not null default now()
);
create index idx_files_card on files(card_id);
```

### 2.12 Operations (Sync Log)

```sql
create type entity_type as enum('user','board','section','tab','card','link','cluster','rule','file');

create table operations (
  id            uuid primary key,
  device_id     text not null,
  user_id       uuid not null references users(id) on delete cascade,
  entity        entity_type not null,
  entity_id     uuid not null,
  op_type       text not null,           -- e.g. 'create','update','delete','move','add_tag'
  payload       jsonb not null,          -- minimal delta payload
  version_prev  integer,
  version_next  integer,
  created_at    timestamptz not null default now()
);
create index idx_operations_entity_time on operations(entity, created_at);
create index idx_operations_user_time on operations(user_id, created_at);
```

### 2.x Utility: `set_updated_at()` trigger function

```sql
create or replace function set_updated_at() returns trigger as $$
begin
  new.updated_at := now();
  return new;
end; $$ language plpgsql;
```

------

## 3) SQLite (Local Mirror) Schema

> SQLite lacks enums & trigram; use `text` for enums, and application‑side guards. JSON stored as `text` or `json` (SQLite JSON1). FTS via `virtual table ... using fts5`.

### 3.1 Conventions

- `id` as `text` (UUID v4 string).
- Timestamps as ISO8601 `text`.
- Enums as constrained `check` expressions.
- Foreign keys enforced with `PRAGMA foreign_keys = ON`.

### 3.2 Tables (mirroring Postgres)

Example: `cards`

```sql
create table cards (
  id           text primary key,
  board_id     text not null references boards(id) on delete cascade,
  section_id   text references sections(id) on delete set null,
  tab_id       text references tabs(id) on delete set null,
  type         text not null check (type in ('note','task','event','drawing')),
  title        text,
  content      json not null default ('{}'),
  ink_data     json,
  due_date     text,
  start_time   text,
  end_time     text,
  priority     integer,
  status       text not null check (status in ('open','in_progress','blocked','done','archived')),
  created_by   text references users(id) on delete set null,
  assignee_id  text references users(id) on delete set null,
  created_at   text not null,
  updated_at   text not null,
  version      integer not null default 0
);
create index idx_cards_board on cards(board_id);
create index idx_cards_section on cards(section_id);
create index idx_cards_due on cards(due_date);
```

Tags & assignees (normalized):

```sql
create table tags (
  id   text primary key,
  name text not null unique
);
create table card_tags (
  card_id text not null references cards(id) on delete cascade,
  tag_id  text not null references tags(id) on delete cascade,
  primary key (card_id, tag_id)
);
create table card_assignees (
  card_id text not null references cards(id) on delete cascade,
  user_id text not null references users(id) on delete cascade,
  primary key (card_id, user_id)
);
```

> Other tables mirror Postgres (fields adapted to `text/json`).

### 3.3 SQLite FTS5

```sql
create virtual table fts_cards using fts5(
  title,
  content_text,
  tokenize = 'porter'
);
-- Application keeps fts_cards in sync from cards.content/title (recognized text).
```

### 3.4 Local Operations Log

```sql
create table operations (
  id            text primary key,
  device_id     text not null,
  user_id       text not null,
  entity        text not null,
  entity_id     text not null,
  op_type       text not null,
  payload       json not null,
  version_prev  integer,
  version_next  integer,
  created_at    text not null
);
create index idx_operations_entity_time on operations(entity, created_at);
```

------

## 4) Canonical JSON (Source of Truth)

> API responses and export/import use these shapes. Keys are **camelCase**. Minimal required fields shown; clients may receive additional fields that should be ignored safely.

### 4.1 Card

```json
{
  "id": "1f2b5e2e-2c7d-4f88-8a8a-0d1e7b5f6b90",
  "boardId": "b5e7...",
  "sectionId": "s9a1...",
  "tabId": null,
  "type": "task",
  "title": "Follow up with Alex",
  "content": {
    "blocks": [
      { "type": "paragraph", "text": "Draft email for proposal." },
      { "type": "checklist", "items": [
        { "text": "Outline", "checked": true },
        { "text": "Review", "checked": false }
      ]}
    ],
    "recognizedText": "draft email for proposal"
  },
  "inkData": { "strokes": [/* vectors */], "snapshotUri": null },
  "dueDate": "2025-10-20T18:00:00Z",
  "startTime": null,
  "endTime": null,
  "priority": 2,
  "status": "open",
  "assignees": ["user-123"],
  "tags": ["work","proposal"],
  "createdBy": "user-456",
  "createdAt": "2025-10-15T10:10:10Z",
  "updatedAt": "2025-10-16T12:00:00Z",
  "version": 4
}
```

### 4.2 Board

```json
{
  "id": "b5e7-...",
  "ownerId": "user-456",
  "title": "Project Delta",
  "visibility": "shared",
  "theme": { "color": "slate", "accent": "teal" },
  "rules": [ { "trigger": "onCardUpdate", "condition": {"field":"content","contains":"meeting"}, "actions": [{"type":"addTag","value":"meeting"}] } ],
  "createdAt": "2025-10-10T09:00:00Z",
  "updatedAt": "2025-10-16T11:30:00Z"
}
```

### 4.3 Section

```json
{
  "id": "s9a1-...",
  "boardId": "b5e7-...",
  "title": "Backlog",
  "position": 1,
  "layoutMeta": { "columns": 1, "collapsed": false },
  "createdAt": "2025-10-10T09:00:00Z",
  "updatedAt": "2025-10-16T11:30:00Z"
}
```

### 4.4 Rule

```json
{
  "id": "r-001",
  "boardId": "b5e7-...",
  "definition": {
    "trigger": "onCardCreate",
    "condition": { "field": "title", "regex": "(?i)meeting" },
    "actions": [ { "type": "suggestConvert", "targetType": "event" } ]
  },
  "enabled": true
}
```

### 4.5 Operation (Delta)

```json
{
  "id": "op-789",
  "deviceId": "android-tablet-001",
  "userId": "user-456",
  "entity": "card",
  "entityId": "1f2b5e2e-2c7d-4f88-8a8a-0d1e7b5f6b90",
  "opType": "update",
  "payload": { "title": "Follow up with Alexandre" },
  "versionPrev": 3,
  "versionNext": 4,
  "createdAt": "2025-10-16T12:00:00Z"
}
```

------

## 5) Search & FTS

### 5.1 Server (Postgres)

- **Generated column** for `tsvector` combining title + recognized text + tags.

```sql
alter table cards add column tsv tsvector generated always as (
  setweight(to_tsvector('simple', coalesce(title,'')), 'A') ||
  setweight(to_tsvector('simple', coalesce((content->>'recognizedText'),'')), 'B')
) stored;
create index idx_cards_tsv on cards using gin(tsv);
```

- Queries combine FTS with metadata filters (board, type, due ranges).

### 5.2 Local (SQLite)

- `fts_cards(title, content_text)` maintained by application service.
- On `cards` change, update or reindex affected doc only.

------

## 6) Sync & Versioning

### 6.1 Version Model

- Each entity has `version` (int) and `updatedAt`.
- Client emits operations with `versionPrev`; server checks and increments.

### 6.2 Delta Endpoints (conceptual)

- `GET /sync?since=cursor` → returns `{ cursor, operations[] }` ordered by `created_at`.
- `POST /sync` → accepts `operations[]`; returns accepted ops + conflicts.

### 6.3 Conflict Policies

- **Scalar fields**: LWW gated by version; server returns conflict if `versionPrev` mismatch.
- **Rich text / ink**: treat as opaque doc; client may use CRDT locally then submit merged content.
- **Positions (sections/tabs)**: store as operation log (`move`, `reorder`); resolve by causal order; ties → last writer.

### 6.4 Undo/Redo

- Client keeps inverse ops for local undo.
- Server can reconstruct via `operations` for audit but not guaranteed to auto-undo (security).

------

## 7) Privacy & E2E Optionality

- Mark card/section as `private=true`.
- Client **encrypts** `content` & `ink_data` with board/user key; server stores ciphertext in JSONB.
- Indexing: private data excluded from server FTS; local FTS still active.

------

## 8) Files / Attachments

- Store binary in object storage (S3/minio); reference via `files.storage_key`.
- Thumbnails & OCR text (if extracted) stored in `files.meta` JSON.
- Link to `cards` and/or `boards`.

------

## 9) Migrations & Schema Versioning

- Maintain `schema_versions` table with ordered migrations.
- Canonical JSON has `"schemaVersion"` field at export root.
- Clients include `dbSchemaVersion` in `/sync` headers for compatibility checks.

------

## 10) Exports & Backups

- **Export**: `.stickyboard` archive → root `manifest.json` (boards, sections, cards, tags, rules, links, activities) + `/files/*`.
- **Import**: id remap stage; detect duplicates via stable content hashes.

------

## 11) Performance & Indexing Guidelines

- Index hot paths: `(board_id, updated_at)`, `(section_id, position)`, `(status, due_date)`, `(created_by)`, FTS `tsv`.
- Avoid large array columns on hot tables; prefer join tables.
- Use pagination (`limit/offset` or keyset via `(updated_at, id)`).
- Consider partitioning `activities` by month for large orgs.

------

## 12) Security & Authorization (Server)

- All writes check `permissions.role` on `board_id`.
- Row‑level filters enforce board membership for Reads.
- `operations.user_id` must match authenticated user.
- Audit everything in `activities`; sensitive data redaction allowed in payload.

------

## 13) Open Questions / Extensibility

- Custom card types: store renderer hints in `content.type`. Validate with JSON Schema.
- Rule DSL registry: server persists definitions; clients execute locally.
- Cluster membership: recompute server‑side periodically; client may maintain local preview.

------

## 14) Minimal JSON Schemas (Draft‑like, indicative)

### 14.1 Card Schema (excerpt)

```json
{
  "$id": "https://aedev.pro/schemas/card.json",
  "type": "object",
  "required": ["id","boardId","type","createdAt","updatedAt","version"],
  "properties": {
    "id": {"type": "string", "format": "uuid"},
    "boardId": {"type": "string", "format": "uuid"},
    "sectionId": {"type": ["string","null"], "format": "uuid"},
    "tabId": {"type": ["string","null"], "format": "uuid"},
    "type": {"enum": ["note","task","event","drawing"]},
    "title": {"type": ["string","null"]},
    "content": {"type": "object"},
    "inkData": {"type": ["object","null"]},
    "dueDate": {"type": ["string","null"], "format": "date-time"},
    "startTime": {"type": ["string","null"], "format": "date-time"},
    "endTime": {"type": ["string","null"], "format": "date-time"},
    "priority": {"type": ["integer","null"]},
    "status": {"enum": ["open","in_progress","blocked","done","archived"]},
    "assignees": {"type": "array", "items": {"type": "string"}},
    "tags": {"type": "array", "items": {"type": "string"}},
    "createdBy": {"type": ["string","null"], "format": "uuid"},
    "createdAt": {"type": "string", "format": "date-time"},
    "updatedAt": {"type": "string", "format": "date-time"},
    "version": {"type": "integer"}
  }
}
```

### 14.2 Board Schema (excerpt)

```json
{
  "$id": "https://aedev.pro/schemas/board.json",
  "type": "object",
  "required": ["id","ownerId","title","visibility","createdAt","updatedAt"],
  "properties": {
    "id": {"type": "string", "format": "uuid"},
    "ownerId": {"type": "string", "format": "uuid"},
    "title": {"type": "string"},
    "visibility": {"enum": ["private","shared","public"]},
    "theme": {"type": "object"},
    "rules": {"type": "array"},
    "createdAt": {"type": "string", "format": "date-time"},
    "updatedAt": {"type": "string", "format": "date-time"}
  }
}
```

------

### End of Spec