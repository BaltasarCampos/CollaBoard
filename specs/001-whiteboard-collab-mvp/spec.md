# Feature Specification: Whiteboard Collaboration MVP

**Feature Branch**: `001-whiteboard-collab-mvp`  
**Created**: 2026-05-09  
**Status**: Draft  
**Input**: User description: "build the mvp for a whiteboard collaboration app. Interface design has to be responsive and simple. All user-facing interfaces MUST provide consistent, predictable, and intuitive experiences. UX patterns MUST be reused. All components MUST meet defined performance budgets. Performance is not a 'nice-to-have'; it directly impacts user experience and operational cost. Minimize payload where possible and prioritize lightweight communication with server. Canvas must respond to drawing in real time and should not wait for the stroke to be finished to render it."

---

## User Scenarios & Testing *(mandatory)*

<!--
  Stories are ordered by priority. Each story is independently testable.
  P1 stories together form the minimum viable product.
-->

### User Story 1 — Room Creation and Joining (Priority: P1)

A new or returning user arrives at the landing page. They enter their display name (required) and optionally a room code. If they leave the room field empty, the system creates a new room and places them inside. If they enter a room code that doesn't exist, the system creates it. If the room code already exists, the user joins that existing room and immediately sees any previously drawn content on the canvas.

**Why this priority**: Without room management there is no shared context —
collaboration cannot begin. All other features depend on this being delivered first.

**Independent Test**: Navigate to the app in a fresh browser tab. Enter user name and create a room.
A second user submitting the same room code should land in the same room. Confirm the second tab connects to the same canvas state.

**Acceptance Scenarios**:

1. **Given** the landing page is open, **When** a user submits a non-empty username and leaves the room field blank,
   **Then** a new room is created, the user is placed inside it, and the whiteboard canvas view is shown.

2. **Given** the landing page is open, **When** a user submits a non-empty username and a room code that does not exist, **Then** a new room with that code is created and the user is placed inside it.

3. **Given** a room already exists, **When** a second user submits their username with that room's code, **Then** the user is added to the existing room, both users appear in the participants panel, and the existing canvas content is replayed to the new user.

4. **Given** a room contains strokes drawn by previous users,
   **When** a new user joins, **Then** all committed strokes are rendered on
   the new user's canvas before any new drawing events are processed.

5. **Given** the landing page form, **When** a user attempts to submit with an empty username, **Then** the form displays a validation error and submission is blocked.

6. **Given** the landing page, **When** the page is viewed on a mobile device, **Then** the form is fully usable and legible without horizontal scrolling.

---

### User Story 2 — Real-Time Freehand Drawing (Priority: P1)

A user opens a collaborative session and draws freehand strokes on the shared
canvas. Every point of a stroke is rendered locally without waiting for the
server round-trip, and other users in the same room see the stroke
appear incrementally — point by point — in real time.

**Why this priority**: Drawing is the core value proposition of the app. Without
real-time, incremental drawing there is no product.

**Independent Test**: Open two browser tabs in the same room. Draw a line in
tab A. Verify that tab B renders each segment of the line as the pointer moves,
not after the pointer is released.

**Acceptance Scenarios**:

1. **Given** a user has joined a room, **When** they press and drag the pointer
   on the canvas, **Then** stroke segments appear on their own canvas immediately
   (≤16 ms per frame — 60 fps budget) without waiting for server acknowledgment.

2. **Given** two users are in the same room, **When** user A draws a stroke,
   **Then** user B sees each incremental segment within 100 ms of it being
   drawn under normal network conditions (≤150 ms RTT).

3. **Given** a stroke is in progress, **When** the pointer is lifted,
   **Then** the final committed stroke is broadcast as a single idempotent
   operation with a unique `operationId`.

4. **Given** a stroke event is received twice (network retry scenario),
   **When** the client processes it, **Then** the canvas state is identical
   to processing it once (idempotency guarantee).

---

### User Story 3 — Participants Panel (Priority: P3)


The whiteboard view includes a visible panel listing all users currently in the room. The panel updates in real time as users join or leave.


**Why this priority**: Presence awareness is a key collaboration feature that builds trust in real-time sync. It can be tested standalone by having users join and leave a room.


**Independent Test**: With two users in a room, confirm both names appear in the participants panel. Have one user close the tab; confirm their name disappears from the other user's panel within a few seconds.


**Acceptance Scenarios**:


1. **Given** a user enters a room, **When** the whiteboard view loads, **Then** the participants panel displays all currently connected users including themselves.
2. **Given** a user is in a room, **When** a new user joins, **Then** the new user's name appears in the participants panel without a page refresh.
3. **Given** a user is in a room, **When** a user disconnects (closes tab or browser), **Then** that user's name is removed from the participants panel within 5 seconds.


---


### User Story 4 — Connection Status Feedback (Priority: P2)

The UI always reflects the user's current connection state (connected,
reconnecting, disconnected) so they are never confused about whether their
actions are being synchronized.

**Why this priority**: Silent failures break user trust. Connection status
is a safety net required by the Resilience principle.

**Independent Test**: Disable the network interface while in a room. Verify a
"Reconnecting…" indicator appears immediately. Re-enable the network. Verify
the indicator changes to "Connected" and canvas state is re-hydrated.

**Acceptance Scenarios**:

1. **Given** the app is connected, **When** the user is in a room,
   **Then** a non-intrusive "Connected" status indicator is visible.

2. **Given** the WebSocket connection drops, **When** the client detects it,
   **Then** an amber "Reconnecting…" indicator appears within 500 ms and
   the client begins exponential-backoff reconnection.

3. **Given** reconnection succeeds, **When** the socket reconnects,
   **Then** the client re-hydrates canvas state from the server and the
   status indicator shows "Connected".

4. **Given** reconnection attempts fail for > 30 seconds, **When** the
   user is still on the page, **Then** the indicator shows "Disconnected"
   and offers a manual "Retry" action.

---

### User Story 5 — Tool Selection: Pen and Eraser (Priority: P3)

The user can switch between a Pen tool (draws strokes) and an Eraser tool
(removes strokes they interact with) via a simple, consistently placed toolbar.

**Why this priority**: Two-tool parity is the minimum ergonomic baseline.
Additional tools (shapes, text) are post-MVP.

**Independent Test**: Draw a stroke. Switch to Eraser. Click/drag over the
stroke. Confirm it is removed.

**Acceptance Scenarios**:

1. **Given** the Pen tool is selected (default on load), **When** the user
   draws, **Then** a new freehand stroke is created.

2. **Given** the Eraser tool is selected, **When** the user drags over a
   stroke, **Then** the strokes intersected are removed and the removal is
   broadcast to all users in the room.

3. **Given** a tool is active, **When** the user looks at the toolbar,
   **Then** the active tool is visually distinct (highlighted/selected state).

---

### Edge Cases

- What happens when a user submits a username that is just whitespace? → Treated 
  as an empty username; form validation blocks submission.
- What happens when two users submit the same display name in the same room? → Both
  are allowed; they are distinguished internally by a session identifier.
- What happens when a user refreshes the page while in a room? → The user is 
  returned to the landing page (no persistent session across refreshes for MVP). 
  If they were the last user, the room is destroyed immediately.
- What happens when a room code contains special characters? → Room codes are 
  sanitized or restricted to alphanumeric characters to prevent injection and 
  routing issues.
- What happens when a user draws during a reconnection window? → Strokes drawn
  offline MUST be queued locally and flushed — with their original `operationId`
  — once the socket reconnects, preventing data loss.
- What happens when two users erase the same stroke simultaneously? → Both
  operations carry the same target `strokeId`; deduplication on the server
  ensures the stroke is removed exactly once (idempotent delete).
- What happens when the canvas has hundreds of strokes and a new user joins? →
  State hydration MUST be chunked or delta-compressed if the payload exceeds a
  configurable threshold (default 50 KB) to keep join latency within the 1 000 ms
  budget.
- What happens on a mobile device with a touch screen? → Pointer events MUST
  handle `touchstart`, `touchmove`, `touchend` equivalently to mouse events;
  the canvas MUST be touch-scrollable when no tool is active.
- What happens when the browser tab is hidden (Page Visibility API)? → Cursor
  events MUST be suspended when the tab is hidden to avoid unnecessary payload.
- What happens when a stroke `operationId` is replayed (retry)? → The server
  deduplicates by `operationId`; the client discards self-originated events
  received via broadcast.

---

## Requirements *(mandatory)*

### Functional Requirements

#### Canvas & Drawing

- **FR-001**: The system MUST render each stroke segment to the local canvas
  immediately upon pointer-move input, with no dependency on server
  acknowledgment (optimistic local rendering).
- **FR-002**: The system MUST broadcast each pointer-move delta as a lightweight
  `stroke:point` event `{ operationId, strokeId, x, y }` to all
  room users.
- **FR-003**: Receiving clients MUST append each incoming `stroke:point` to the
  in-progress stroke and render it to the shared canvas layer within one
  animation frame.
- **FR-004**: On pointer-up, the system MUST emit a `stroke:commit` event
  `{ operationId, strokeId, color, width, points[] }` finalizing the stroke.
  This is the idempotent, durable record of the stroke.
- **FR-005**: The server MUST deduplicate `stroke:commit` events by `operationId`
  before broadcasting to the room.
- **FR-006**: The system MUST support erasing strokes by `strokeId` via an
  `stroke:erase` event; erasing is idempotent.

#### Room Management

- **FR-008**: The system MUST allow a user to create a new room; the server
  MUST return a unique room code.
- **FR-009**: The system MUST allow a user to join an existing room by room code;
  on join the server MUST send the current committed canvas state
  (`room:hydrate` event).
- **FR-010**: The server MUST maintain room state in memory; rooms with no 
  users MUST be inmediatly deleted.
- **FR-011**: The system MUST provide a landing page with a form containing a 
  mandatory username field and an optional room code field.
- **FR-012**: The system MUST validate that the username field is non-empty and
  non-whitespace before allowing form submission.
- **FR-013**: When a room code is not provided, the system MUST automatically
  generate a unique room identifier and create a new room.
- **FR-014**: When a provided room code does not correspond to an existing room,
  the system MUST create a new room with that code and place the user inside it.

#### Collaboration & Sync

- **FR-015**: The client MUST queue locally drawn strokes (not yet acknowledged)
  during a connection outage and replay them on reconnect, preserving original
  `operationId` values.

#### Undo

- **FR-016**: The system MUST support per-user undo of the most recent committed
  stroke via `stroke:undo` `{ operationId, strokeId, userId }`.
- **FR-017**: Undo MUST only affect strokes owned by the requesting `userId`.
- **FR-018**: The Undo button/shortcut MUST be disabled when the user has no
  undoable strokes.

#### Connection Status

- **FR-019**: The client MUST display connection status at all times:
  `connected | reconnecting | disconnected`.
- **FR-020**: On WebSocket disconnection, the client MUST begin exponential
  backoff reconnection (initial 500 ms, max 30 s, jitter ±20%).
- **FR-021**: On successful reconnection, the client MUST re-request canvas
  state via a `room:rejoin` event before processing any new operations.

#### Performance Budgets

- **FR-022**: Local stroke rendering latency MUST be ≤16 ms per frame (60 fps).
- **FR-023**: Remote stroke segment delivery MUST be ≤150 ms P99 under ≤150 ms
  RTT network conditions.
- **FR-024**: Room join/hydration MUST complete within 1 000 ms for canvases
  with ≤500 committed strokes.
- **FR-025**: Each `stroke:point` payload MUST be ≤128 bytes (binary/JSON).
- **FR-026**: The client JavaScript bundle (initial load) MUST be ≤250 KB
  gzipped.
- **FR-027**: Time-to-interactive for the whiteboard room page MUST be ≤3 s on
  a simulated 4G connection.

#### Responsive & Accessible UI

- **FR-028**: The UI MUST be functional and visually consistent on viewport
  widths from 375 px (mobile) to 1 920 px (desktop).
- **FR-029**: The toolbar MUST reuse a single `ToolButton` component pattern
  across all tool entries; no one-off button implementations.
- **FR-030**: The connection status indicator MUST reuse the same visual
  component wherever status is surfaced.
- **FR-031**: Interactive controls MUST meet WCAG 2.1 AA minimum contrast and
  have accessible labels.

### Key Entities

- **Room**: Identified by a unique room code. Holds the ordered list
  of committed strokes and the set of connected users.
- **Stroke**: Identified by `strokeId` (UUID v4). Owned by a `userId`. Contains
  an ordered array of `{x, y}` points, `color`, `width`, and `operationId`.
- **StrokePoint**: An incremental delta `{strokeId, x, y}` used
  only in transit (`stroke:point` event); never persisted as a standalone entity.
- **User**: A connected socket identified by `userId` and display name.
   Not persisted beyond the active session for MVP.
- **Operation**: Any mutation of canvas state, identified by `operationId`
  (UUID v4). Types: `stroke:commit`, `stroke:erase`, `stroke:undo`, `canvas:clear`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Local stroke rendering runs at a sustained 60 fps (≤16 ms per
  frame) on a mid-range device while drawing continuously for 60 seconds.
- **SC-002**: A second user receives and renders each stroke segment
  within 150 ms P99 of it being drawn, measured on a simulated 100 ms RTT
  link.
- **SC-003**: A new user joining a room with 500 existing strokes has the
  full canvas hydrated and ready within 1 000 ms of the socket connecting.
- **SC-004**: Each individual `stroke:point` Socket.IO message payload is
  ≤128 bytes; each `cursor:move` payload is ≤64 bytes.
- **SC-005**: The gzipped client bundle is ≤250 KB; time-to-interactive on a
  simulated 4G connection is ≤3 s.
- **SC-006**: Replaying the same `stroke:commit` event 10 consecutive times
  produces an identical canvas state to processing it once.
- **SC-007**: All committed strokes drawn in a room are visible to a user who
  joins after a WebSocket disconnect and reconnect cycle.
- **SC-008**: The UI renders and is usable at viewport widths of 375 px, 768 px,
  and 1 440 px without horizontal scrolling or layout breakage.
- **SC-009**: A user who draws strokes during a 5-second simulated network
  outage has all those strokes appear on the canvas (for all users)
  within 3 s of reconnection.
- **SC-010**: The `ToolButton` component is the sole implementation used for
  every toolbar button; zero one-off button variants exist in the codebase.

---

## Assumptions

- Users do not require persistent accounts or authentication in the MVP; any
  user who knows the room code may join the room.
- Room state is stored in server memory only (no database); data loss on server
  restart is acceptable for the MVP.
- The MVP targets modern evergreen browsers (Chrome, Firefox, Edge, Safari
  latest two versions); IE and legacy browsers are out of scope.
- Mobile support means a responsive layout and touch-event handling; a native
  mobile app is out of scope.
- Scalability beyond a single Node.js process (e.g., horizontal scaling with
  Redis adapter) is out of scope for the MVP but the architecture MUST NOT
  prevent it.
- Text, shapes, images, and sticky-note tools are out of scope for the MVP.
- Export/download of canvas content is out of scope for the MVP.
- Canvas zoom (pinch-to-zoom) is a stretch goal.
- Stroke width and color selection defaults to a single pen style in the MVP;
  a color/width picker is a stretch goal.
- The room code generated automatically is a short, shareable alphanumeric 
  string (e.g., 6 characters); its exact format is an implementation detail.
- The drawing canvas uses the HTML5 Canvas 2D context. Canvas state is represented
  as an ordered list of stroke commands, enabling full replay to late joiners and 
  efficient clear operations.
- No moderation, content filtering, or abuse prevention features are required for MVP.

