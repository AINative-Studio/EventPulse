# EventPulse Data Model (ZeroDB)

## 0) Conventions

* **IDs**: UUID v4 strings.
* **Timestamps**: ISO 8601 (`2025-08-13T09:15:27.123Z`).
* **Soft deletes**: `deleted_at` nullable timestamp.
* **Doc metadata**: every doc has `created_at`, `updated_at`, `created_by`, `updated_by`, `version` (int, optimistic concurrency).
* **Multi-tenant keys**: store `org_id` if you’ll need it later (optional now).
* **Indexes**: specified per collection.

---

## 1) Users

> Source of truth may be your auth provider; store a lightweight user profile for display and mentions.

**Collection:** `users`

```json
{
  "user_id": "uuid",
  "email": "string",
  "display_name": "string",
  "avatar_url": "string|null",
  "status": "active|disabled",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid|null",
  "updated_by": "uuid|null",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `idx_users_email` (unique)
* `idx_users_status`

---

## 2) Boards & Membership

**Collection:** `boards`

```json
{
  "board_id": "uuid",
  "name": "string",
  "owner_id": "uuid",
  "visibility": "private|link",            // "link" = shared via URL token
  "share_token": "string|null",            // when visibility=link
  "settings": {
    "wip_limits": { "column_id": "int" },  // optional
    "color": "string|null",                // board theme
    "allow_guests": true
  },
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `idx_boards_owner_id`
* `idx_boards_visibility`
* `idx_boards_share_token` (unique, nullable)

**Collection:** `board_members`

```json
{
  "board_member_id": "uuid",
  "board_id": "uuid",
  "user_id": "uuid",
  "role": "owner|editor|viewer",
  "invited_by": "uuid|null",
  "invited_at": "timestamp|null",
  "joined_at": "timestamp|null",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `idx_board_members_board_id`
* `idx_board_members_user_id`
* `(board_id, user_id)` unique

**Access Control (denormalized helper):** `board_acl`

```json
{
  "board_id": "uuid",
  "acl": [
    { "user_id": "uuid", "role": "owner|editor|viewer" }
  ],
  "updated_at": "timestamp",
  "version": 1
}
```

*Used for quick RBAC checks client-side.*

---

## 3) Columns (Swimlanes)

> MVP has three columns, but make it configurable.

**Collection:** `board_columns`

```json
{
  "column_id": "uuid",
  "board_id": "uuid",
  "name": "string",                  // e.g., "Todo", "In Progress", "Done"
  "order": 0,                        // integer sort
  "wip_limit": 0,                    // 0 = unlimited
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `(board_id, order)`
* `idx_columns_board_id`

---

## 4) Tasks

**Collection:** `tasks`

```json
{
  "task_id": "uuid",
  "board_id": "uuid",
  "column_id": "uuid",
  "title": "string",
  "description": "string|null",
  "priority": "low|medium|high|urgent",
  "position": 0,                         // for intra-column ordering
  "assignees": ["uuid"],                 // multiple users
  "labels": ["uuid"],                    // references task_labels.label_id
  "due_at": "timestamp|null",
  "attachments_count": 0,
  "comments_count": 0,
  "checklist_summary": {                 // denormalized for speed
    "items": 0,
    "completed": 0
  },
  "activity_seq": 0,                     // last applied event seq for idempotency
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 3,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `idx_tasks_board_id`
* `(board_id, column_id, position)`
* `(board_id, labels[])` (array index, if supported)
* `(board_id, assignees[])` (array index, if supported)
* `idx_tasks_due_at`

---

## 5) Labels/Tags

**Collection:** `task_labels`

```json
{
  "label_id": "uuid",
  "board_id": "uuid",
  "name": "string",
  "color": "string",            // e.g. hex
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `(board_id, name)` unique

---

## 6) Comments

**Collection:** `task_comments`

```json
{
  "comment_id": "uuid",
  "task_id": "uuid",
  "board_id": "uuid",
  "author_id": "uuid",
  "body": "string",
  "mentions": ["uuid"],                 // optional
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `idx_comments_task_id`
* `(board_id, task_id, created_at)`

---

## 7) Attachments (stub for later)

**Collection:** `task_attachments`

```json
{
  "attachment_id": "uuid",
  "task_id": "uuid",
  "board_id": "uuid",
  "uploader_id": "uuid",
  "file_name": "string",
  "file_size": 0,
  "mime_type": "string",
  "storage_url": "string",              // pre-signed or permanent
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `idx_attachments_task_id`

---

## 8) Checklists (optional but easy win)

**Collection:** `task_checklists`

```json
{
  "checklist_id": "uuid",
  "task_id": "uuid",
  "board_id": "uuid",
  "title": "string|null",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Collection:** `task_checklist_items`

```json
{
  "item_id": "uuid",
  "checklist_id": "uuid",
  "task_id": "uuid",
  "board_id": "uuid",
  "body": "string",
  "checked": false,
  "position": 0,
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "created_by": "uuid",
  "updated_by": "uuid",
  "version": 1,
  "deleted_at": "timestamp|null"
}
```

**Indexes**

* `(task_id, position)`
* `idx_checklist_items_checklist_id`

---

## 9) Presence (ephemeral)

**Collection:** `presence_sessions`

```json
{
  "session_id": "uuid",
  "board_id": "uuid",
  "user_id": "uuid",
  "client_id": "string",                 // device/tab identifier
  "status": "online|offline",
  "cursor": { "x": 0, "y": 0 },          // optional for future cursors
  "last_seen_at": "timestamp",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "version": 1,
  "expires_at": "timestamp"              // TTL for auto-cleanup
}
```

**Indexes**

* `(board_id, user_id)`
* `idx_presence_expires_at` (TTL)

> Presence state is **primarily event-driven**; this collection is for sanity checks & cold-start rendering.

---

## 10) Activity Log (append-only)

**Collection:** `board_activity`

```json
{
  "activity_id": "uuid",
  "board_id": "uuid",
  "actor_id": "uuid",
  "event_type": "task_created|task_updated|task_moved|task_deleted|comment_added|label_added|label_removed|assignee_added|assignee_removed|column_created|column_renamed|column_deleted|user_joined|user_left",
  "payload": { "any": "json" },         // snapshot of change for audit
  "seq": 0,                             // monotonic per board (see events)
  "created_at": "timestamp"
}
```

**Indexes**

* `(board_id, seq)` unique
* `(board_id, created_at)`

---

## 11) Event Envelope (for publish & replay)

> You’ll **publish** these to `/events/publish` and **consume** on `/events/stream`.
> Persist a normalized copy to enable rewind/replay and idempotency.

**Collection:** `events`

```json
{
  "event_id": "uuid",
  "board_id": "uuid",
  "channel": "string",                   // e.g. "board_{board_id}"
  "event_type": "string",
  "actor_id": "uuid",
  "payload": { "any": "json" },
  "seq": 0,                              // per-board sequence number
  "correlation_id": "uuid|null",         // cross-event correlation
  "causation_id": "uuid|null",           // producer event lineage
  "occurred_at": "timestamp",
  "received_at": "timestamp",
  "checksum": "string|null",             // optional integrity check
  "version": 1
}
```

**Indexes**

* `(board_id, seq)` unique
* `idx_events_channel`
* `idx_events_event_type`

**Channel convention**

* `board_{board_id}` (all events for a board)
* (Optional future) `board_{board_id}_task_{task_id}` for granular subscriptions

---

## 12) Client Outbox (offline/sync)

**Collection:** `client_outbox`

```json
{
  "outbox_id": "uuid",
  "client_id": "string",                 // browser tab/device
  "user_id": "uuid",
  "board_id": "uuid",
  "pending_event": {
    "event_type": "string",
    "payload": { "any": "json" }
  },
  "status": "queued|publishing|published|failed",
  "retries": 0,
  "last_error": "string|null",
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

**Indexes**

* `(client_id, status)`
* `idx_outbox_board_id`

---

# Event Types & Payload Contracts

> Keep them stable; version if you need breaking changes.

### 1) `task_created`

```json
{
  "task": {
    "task_id": "uuid",
    "board_id": "uuid",
    "column_id": "uuid",
    "title": "string",
    "description": "string|null",
    "priority": "low|medium|high|urgent",
    "position": 0,
    "assignees": ["uuid"],
    "labels": ["uuid"],
    "due_at": "timestamp|null"
  }
}
```

### 2) `task_updated`

```json
{
  "task_id": "uuid",
  "changes": {
    "title": "string|undefined",
    "description": "string|null|undefined",
    "priority": "low|medium|high|urgent|undefined",
    "assignees": ["uuid"] ,
    "labels": ["uuid"],
    "due_at": "timestamp|null|undefined"
  }
}
```

### 3) `task_moved`

```json
{
  "task_id": "uuid",
  "from": { "column_id": "uuid", "position": 3 },
  "to":   { "column_id": "uuid", "position": 0 }
}
```

### 4) `task_deleted`

```json
{
  "task_id": "uuid"
}
```

### 5) `comment_added`

```json
{
  "task_id": "uuid",
  "comment": {
    "comment_id": "uuid",
    "author_id": "uuid",
    "body": "string",
    "mentions": ["uuid"]
  }
}
```

### 6) `label_added` / `label_removed`

```json
{
  "task_id": "uuid",
  "label_id": "uuid"
}
```

### 7) `assignee_added` / `assignee_removed`

```json
{
  "task_id": "uuid",
  "user_id": "uuid"
}
```

### 8) `column_created` / `column_renamed` / `column_deleted`

```json
{
  "column": {
    "column_id": "uuid",
    "name": "string",
    "order": 0,
    "wip_limit": 0
  }
}
```

### 9) `user_joined` / `user_left` (presence)

```json
{
  "user_id": "uuid",
  "session_id": "uuid"
}
```

---

# Index Strategy (ZeroDB)

* **Boards:** by owner, visibility, token.
* **Tasks:** composite `(board_id, column_id, position)` for fast lane rendering; additional array indexes for `assignees` and `labels` if supported.
* **Activity/Events:** `(board_id, seq)` for replay and causality.
* **Presence:** TTL on `expires_at`.

> If ZeroDB supports **materialized views**, create `tasks_by_board_column` and `activity_recent_by_board (limit 50)`.

---

# Optimistic Concurrency & Idempotency

* Each mutable doc has a `version` integer.
* On update: `if_match_version` must equal stored `version`, then increment.
* Event consumers track last `seq` per board; drop/ignore events with `seq` ≤ last seen.
* For exactly-once UI application, store `task.activity_seq` = last applied `seq`.

**Collection:** `board_sequences`

```json
{
  "board_id": "uuid",
  "last_seq": 0,
  "updated_at": "timestamp"
}
```

---

# Security Model (RBAC)

* **Roles** per `board_members`:

  * `owner`: CRUD boards, columns, tasks, members; delete board.
  * `editor`: CRUD tasks/columns (no member mgmt).
  * `viewer`: read-only; can’t publish mutating events.
* **Auth propagation**: include `user_id` & signed token when calling `/events/publish`. Client enforces role before emitting; server can validate with ZeroDB auth hooks if available.

---

# Minimal Seed Data (MVP)

1. **Columns**

```json
[
  { "column_id": "uuid1", "board_id": "B", "name": "Todo",        "order": 0, "wip_limit": 0 },
  { "column_id": "uuid2", "board_id": "B", "name": "In Progress", "order": 1, "wip_limit": 3 },
  { "column_id": "uuid3", "board_id": "B", "name": "Done",        "order": 2, "wip_limit": 0 }
]
```

2. **Event channel**

* `board_B` for all subscribers on board `B`.

---

# Optional: Parallel SQL DDL (if you mirror to Postgres)

```sql
CREATE TABLE boards (
  board_id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  owner_id UUID NOT NULL,
  visibility TEXT NOT NULL CHECK (visibility IN ('private','link')),
  share_token TEXT UNIQUE,
  settings JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL,
  created_by UUID,
  updated_by UUID,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ
);

CREATE TABLE board_members (
  board_member_id UUID PRIMARY KEY,
  board_id UUID NOT NULL REFERENCES boards(board_id) ON DELETE CASCADE,
  user_id UUID NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('owner','editor','viewer')),
  invited_by UUID,
  invited_at TIMESTAMPTZ,
  joined_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL,
  created_by UUID,
  updated_by UUID,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ,
  UNIQUE (board_id, user_id)
);

CREATE TABLE board_columns (
  column_id UUID PRIMARY KEY,
  board_id UUID NOT NULL REFERENCES boards(board_id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  "order" INT NOT NULL,
  wip_limit INT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL,
  created_by UUID,
  updated_by UUID,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_board_columns_board_order ON board_columns(board_id, "order");

CREATE TABLE tasks (
  task_id UUID PRIMARY KEY,
  board_id UUID NOT NULL REFERENCES boards(board_id) ON DELETE CASCADE,
  column_id UUID NOT NULL REFERENCES board_columns(column_id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  priority TEXT NOT NULL CHECK (priority IN ('low','medium','high','urgent')),
  position INT NOT NULL,
  assignees UUID[] NOT NULL DEFAULT '{}',
  labels UUID[] NOT NULL DEFAULT '{}',
  due_at TIMESTAMPTZ,
  attachments_count INT NOT NULL DEFAULT 0,
  comments_count INT NOT NULL DEFAULT 0,
  checklist_summary JSONB NOT NULL DEFAULT '{"items":0,"completed":0}',
  activity_seq BIGINT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL,
  created_by UUID,
  updated_by UUID,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_tasks_board_column_pos ON tasks(board_id, column_id, position);
CREATE INDEX idx_tasks_board ON tasks(board_id);
CREATE INDEX idx_tasks_due_at ON tasks(due_at);

CREATE TABLE task_labels (
  label_id UUID PRIMARY KEY,
  board_id UUID NOT NULL REFERENCES boards(board_id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  color TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL,
  created_by UUID,
  updated_by UUID,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ,
  UNIQUE (board_id, name)
);

CREATE TABLE task_comments (
  comment_id UUID PRIMARY KEY,
  task_id UUID NOT NULL REFERENCES tasks(task_id) ON DELETE CASCADE,
  board_id UUID NOT NULL REFERENCES boards(board_id) ON DELETE CASCADE,
  author_id UUID NOT NULL,
  body TEXT NOT NULL,
  mentions UUID[] NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL,
  updated_at TIMESTAMPTZ NOT NULL,
  created_by UUID,
  updated_by UUID,
  version INT NOT NULL DEFAULT 1,
  deleted_at TIMESTAMPTZ
);

CREATE TABLE board_activity (
  activity_id UUID PRIMARY KEY,
  board_id UUID NOT NULL REFERENCES boards(board_id) ON DELETE CASCADE,
  actor_id UUID NOT NULL,
  event_type TEXT NOT NULL,
  payload JSONB NOT NULL,
  seq BIGINT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  UNIQUE (board_id, seq)
);

CREATE TABLE events (
  event_id UUID PRIMARY KEY,
  board_id UUID NOT NULL REFERENCES boards(board_id) ON DELETE CASCADE,
  channel TEXT NOT NULL,
  event_type TEXT NOT NULL,
  actor_id UUID NOT NULL,
  payload JSONB NOT NULL,
  seq BIGINT NOT NULL,
  correlation_id UUID,
  causation_id UUID,
  occurred_at TIMESTAMPTZ NOT NULL,
  received_at TIMESTAMPTZ NOT NULL,
  checksum TEXT,
  version INT NOT NULL DEFAULT 1,
  UNIQUE (board_id, seq)
);
```

---

# Data Lifecycle & Retention

* **Events**: keep 90 days (configurable). Archive older to cold storage; retain per-board `last_seq`.
* **Activity**: store last 5k per board (or 50 for compact UI); older roll-up to daily summaries.
* **Presence**: TTL 60–120 seconds after `last_seen_at`; periodic sweeper job (client can also publish heartbeats).

---

# Mapping Real‑Time → Storage

* **On publish**: client emits event → server(less)/edge forwards to ZeroDB `/events/publish` and (optionally) writes the normalized doc update and the `events` record (with `seq`) to ZeroDB documents.
* **On stream**: client applies in-memory; if optimistic update already applied, reconcile using `seq` & `version`.

---
