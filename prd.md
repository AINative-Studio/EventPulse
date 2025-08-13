# **Product Requirements Document (PRD)**

**Project Name:** EventPulse
**Tagline:** Real-time collaborative task boards made simple.
**Version:** 1.0
**Created On:** 2025-08-13

---

## **1. Executive Summary**

EventPulse is a **lightweight, browser-based real-time task board** for teams to create, assign, and update tasks collaboratively. It is powered by **ZeroDB’s real-time event APIs** (`/events/publish`, `/events/stream`) for instant updates across all connected clients.

The app focuses on **speed, simplicity, and zero-latency collaboration** — no heavy backend beyond ZeroDB’s event streams and a minimal persistence layer for board/task state.

---

## **2. Goals & Objectives**

| Goal                        | Objective                                                                  |
| --------------------------- | -------------------------------------------------------------------------- |
| ⚡ Instant task updates      | Use `/events/stream` to push changes in <200ms to all connected clients.   |
| 🪶 Lightweight architecture | No custom backend for events — just ZeroDB APIs + static hosting.          |
| 👥 Multi-user collaboration | Multiple users can view & edit the same board in real time.                |
| 📝 Offline → Sync           | Allow local task changes to be queued and pushed when connection restores. |
| 📱 Responsive UI            | Same experience on desktop, tablet, and mobile.                            |

---

## **3. Core Features**

### **3.1 Board Management**

* Create, rename, delete task boards.
* Boards identified by a unique `board_id` (UUID).
* Public boards (shareable via URL) and private boards (invite only).

### **3.2 Task Management**

* Create, edit, delete tasks.
* Assign tasks to a user.
* Change task status (columns: Todo, In Progress, Done).
* Reorder tasks via drag-and-drop.

### **3.3 Real-Time Sync (ZeroDB)**

* **Publish changes:**
  When a user edits a task, the frontend sends an event to `/events/publish`:

  ```json
  {
    "channel": "board_{board_id}",
    "event_type": "task_updated",
    "payload": { "task_id": "123", "status": "done" }
  }
  ```
* **Receive changes:**
  Clients subscribe to `/events/stream` for their board’s channel:

  ```json
  {
    "subscribe": ["board_{board_id}"]
  }
  ```
* Apply incoming changes instantly to the UI.

### **3.4 Presence Indicators**

* Show which users are currently viewing the board.
* `/events/publish` presence events when users join/leave.

### **3.5 Activity Log**

* Store and display the last 50 changes on the board.
* Update in real time as new events arrive.

---

## **4. Non-Functional Requirements**

* **Performance:**

  * Event delivery latency: <200ms.
  * Task re-render: <50ms after receiving update.
* **Security:**

  * Auth tokens passed when subscribing/publishing.
  * Role-based access (Owner, Editor, Viewer).
* **Scalability:**

  * Support 500 concurrent users per board.
* **Resilience:**

  * Auto-reconnect to `/events/stream` on disconnect.
  * Offline queue for publish events.

---

## **5. Tech Stack**

| Layer             | Tech                                                      |
| ----------------- | --------------------------------------------------------- |
| Frontend          | React + TailwindCSS                                       |
| Real-time         | ZeroDB `/events/publish`, `/events/stream`                |
| State Persistence | ZeroDB document store (`/documents`) for board/task state |
| Hosting           | Vercel or Netlify                                         |
| Auth              | JWT-based (ZeroDB `/auth`) or external                    |

---

## **6. API Usage**

### **6.1 Publish Event**

**POST** `/events/publish`

```json
{
  "channel": "board_abc123",
  "event_type": "task_created",
  "payload": {
    "task_id": "t1",
    "title": "Design login page",
    "status": "todo",
    "assigned_to": "user_45"
  }
}
```

### **6.2 Stream Events**

**GET** `/events/stream` (SSE or WebSocket)
**Request:**

```json
{
  "subscribe": ["board_abc123"]
}
```

**Incoming Event Example:**

```json
{
  "channel": "board_abc123",
  "event_type": "task_status_changed",
  "payload": {
    "task_id": "t1",
    "status": "done"
  }
}
```

---

## **7. Data Model (ZeroDB Collections)**

**Collection: `boards`**

```json
{
  "board_id": "uuid",
  "name": "string",
  "owner_id": "uuid",
  "members": ["uuid"],
  "created_at": "timestamp"
}
```

**Collection: `tasks`**

```json
{
  "task_id": "uuid",
  "board_id": "uuid",
  "title": "string",
  "description": "string",
  "status": "todo|in_progress|done",
  "assigned_to": "uuid",
  "created_at": "timestamp",
  "updated_at": "timestamp"
}
```

**Collection: `activity_log`**

```json
{
  "log_id": "uuid",
  "board_id": "uuid",
  "event_type": "string",
  "payload": "json",
  "timestamp": "timestamp",
  "user_id": "uuid"
}
```

---

## **8. User Flows**

### **8.1 Create Board**

1. User clicks “New Board.”
2. POST board to ZeroDB `/documents/create`.
3. Auto-subscribe to `/events/stream` for that board.

### **8.2 Update Task**

1. User edits task locally.
2. Send `/events/publish` with `task_updated`.
3. Other clients receive and update UI instantly.

### **8.3 Presence**

1. On board open, publish `user_joined`.
2. On tab close, publish `user_left`.

---

## **9. Milestones & MVP Scope**

**MVP (Day 1)**

* Single board.
* Add/edit/delete tasks.
* Real-time sync with `/events/publish` and `/events/stream`.
* Presence indicators.

**Post-MVP**

* Multiple boards.
* Activity log with filtering.
* File attachments.
* Task comments.

---

