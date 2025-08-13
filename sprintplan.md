## **Sprint Name:** EventPulse MVP – Lightning Build

**Timebox:** 60 minutes
**Goal:** Ship a single-board, real-time collaborative task board with create/move/update task flows, powered by ZeroDB events.

---

### **0. Pre-Sprint Setup (5 min – concurrent if possible)**

* **Tasks:**

  * Clone boilerplate React + Tailwind project (e.g., Vite or Next.js starter).
  * Set up `.env` with ZeroDB API key & endpoint.
  * Install deps:

    ```bash
    npm install axios @tailwindcss/forms uuid
    ```
  * Create `/lib/zerodb.js` with wrappers for:

    * `publishEvent(channel, event_type, payload)`
    * `subscribeToStream(channel, onMessage)`

---

### **1. Data Model & Event Contracts (5 min)**

* Define **in-memory data structures** for board, columns, tasks (mirror the ZeroDB collections but keep minimal for MVP).
* Define **event types**:

  * `task_created`
  * `task_updated`
  * `task_moved`
  * `task_deleted`
  * `user_joined` / `user_left`
* Write **constant channels** (e.g., `BOARD_CHANNEL = "board_{board_id}"`).

---

### **2. UI Scaffolding (15 min)**

* **Board Layout**:

  * Fixed 3 columns: **Todo / In Progress / Done**.
  * Each column = `<Column title="..." tasks={...} />`.
* **Task Card Component**:

  * Title, edit/delete buttons, assignee placeholder.
  * Click → open quick edit modal.
* **New Task Form**:

  * Inline at top of Todo column.
  * On submit → publish `task_created` event.
* **Drag-and-drop**:

  * Use `react-beautiful-dnd` (or skip for MVP, just add move buttons).
  * On move → publish `task_moved` event.

---

### **3. Real-Time Integration (20 min)**

**Incoming events:**

1. Connect to `/events/stream` with board channel.
2. On message:

   * Parse event\_type.
   * Update local state accordingly.
   * Re-render columns.

**Outgoing events:**

1. Create task → `publishEvent("board_123", "task_created", {...})`.
2. Update task → `publishEvent("board_123", "task_updated", {...})`.
3. Move task → `publishEvent("board_123", "task_moved", {...})`.

**Presence:**

* On mount → `publishEvent("board_123", "user_joined", { user_id })`.
* On unmount → `publishEvent("board_123", "user_left", { user_id })`.

---

### **4. Persistence Hook (5 min)**

* For MVP:

  * Store board & tasks in **local state**.
  * Optionally call ZeroDB `/documents` to persist after publish.
* Create `useBoardState` hook:

  * `state.tasks`, `addTask()`, `updateTask()`, `moveTask()`.
  * Wrap publish calls so state & events always match.

---

### **5. Final Polish & Demo Flow (10 min)**

* Add minimal Tailwind styling for clean columns & cards.
* Show presence avatars at top of board.
* Log incoming events to a small **Activity Panel**.
* Test with **two browser tabs**:

  * Tab 1 creates a task → Tab 2 updates instantly.
  * Tab 2 moves task → Tab 1 reflects instantly.
* Push to Vercel/Netlify for live demo.

---

## **Deliverables after 60 minutes**

* One board with:

  * Create/edit/move/delete tasks in real time.
  * Presence indicators.
  * Clean, responsive UI.
* ZeroDB-powered event streaming (no custom backend).
* Ready-to-demo deployment link.

---

## **Time Allocation Recap**

| Phase                  | Time | Outcome                        |
| ---------------------- | ---- | ------------------------------ |
| Pre-Sprint Setup       | 5m   | Env, deps, ZeroDB client ready |
| Data Model & Contracts | 5m   | Event payload shapes fixed     |
| UI Scaffolding         | 15m  | Columns, cards, forms in place |
| Real-Time Integration  | 20m  | Stream/publish wired & tested  |
| Persistence & Polish   | 15m  | Styling, presence, deploy      |

---

If you want, I can follow this up with a **GitHub Issues–ready backlog JSON** that exactly maps to this one-hour sprint so you can drop it into a repo and start coding without writing a single ticket.
