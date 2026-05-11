# Research: Whiteboard Collaboration MVP

**Branch**: `001-whiteboard-collab-mvp` | **Date**: 2026-05-10  
**Purpose**: Resolve all technical unknowns and document decisions before Phase 1 design

---

## 1. Real-Time Freehand Drawing — Canvas Rendering Strategy

**Decision**: Use a **dual-canvas architecture** with a `requestAnimationFrame` (rAF) render loop for the local drawing layer.

**Rationale**:
- A single canvas must be cleared and redrawn on every frame when any state changes. For 500+ committed strokes this becomes expensive.
- A **committed canvas** (bottom layer) holds all finalized `stroke:commit` strokes and is redrawn only when the committed stroke list changes (on commit, erase, undo, or hydration).
- An **in-progress canvas** (top layer, transparent background) holds the currently-drawing stroke and is cleared/redrawn on every `pointermove` via rAF, keeping it strictly at 60 fps.
- Remote in-progress strokes from other users are also rendered on the in-progress layer, cleared each frame and re-drawn from an in-memory point buffer keyed by `strokeId`.
- This avoids React re-renders on every pointer event — the canvas is a DOM element managed imperatively via a `useCanvas` hook and a forwarded `ref`.

**Alternatives considered**:
- *Single canvas with full redraw per frame*: Fails the ≤16 ms/frame budget at high stroke counts.
- *SVG paths*: DOM manipulation overhead is incompatible with 60 fps; SVG path segments accumulate in the DOM.
- *WebGL/OffscreenCanvas*: Achieves better throughput but adds significant complexity out of scope for MVP.

---

## 2. Socket.IO Payload Optimization — `stroke:point` ≤ 128 Bytes

**Decision**: Send `stroke:point` as a **compact JSON object** with 2-decimal-place truncated coordinates; no binary encoding for MVP.

**Rationale**:
- A minimal `stroke:point` payload: `{"operationId":"<uuid-v4>","strokeId":"<uuid-v4>","x":1234.56,"y":789.01}` is approximately 100–110 bytes as UTF-8 JSON, within the 128-byte budget.
- UUID v4 strings are 36 characters each. Two UUIDs cost 72 bytes; the remaining ~50 bytes cover key names, separators, and coordinate values.
- Coordinate values are truncated to 2 decimal places before transmission (sub-pixel precision is imperceptible and saves bytes). A helper `round2(n)` — `Math.round(n * 100) / 100` — is applied before emit.
- If future stroke complexity (e.g., pressure, tilt) pushes payloads beyond 128 bytes, Socket.IO binary framing (`Buffer`) with a fixed binary layout should be adopted. This is documented as a post-MVP upgrade path.

**Alternatives considered**:
- *Binary framing now*: Two 4-byte floats + two 16-byte UUIDs = 40 bytes, well under budget — but adds encoding/decoding complexity.
- *Short IDs instead of UUIDs*: Reduces payload but requires a server-side ID registry, complicating deduplication.

---

## 3. Conflict Resolution & Convergence Strategy

**Decision**: **Server-ordered, append-only log with idempotent writes**; last-receipt-wins for concurrent erases of the same stroke.

**Rationale**:
- The MVP stroke model is append-only: strokes are added (`stroke:commit`) and removed (`stroke:erase`, `stroke:undo`). There are no in-place mutations.
- The server maintains the canonical ordered list. When the server receives a `stroke:commit` it assigns a monotonically increasing server sequence number and broadcasts. Clients replay strokes in this order on hydration.
- Concurrent erases of the same `strokeId` are naturally idempotent — deleting an already-deleted entry is a no-op. The first erase removes the stroke; subsequent erases with the same `strokeId` do nothing.
- Concurrent `stroke:commit` from different users creates entries in server-receipt order, which all clients converge to because hydration replays the server's ordered list.
- This is equivalent to a simple **last-writer-wins** model at the stroke granularity, which is sufficient because strokes are independent atomic units.

**Alternatives considered**:
- *Operational Transform (OT)*: Required when operations can conflict at a fine-grained level (e.g., text editing). Not needed for independent atomic strokes.
- *CRDT (e.g., Yjs)*: Correct but adds a dependency and complexity beyond MVP needs. Documented as a future upgrade if collaborative features expand (e.g., shared text annotations, shape positioning).

---

## 4. State Hydration — Large Canvas Chunking

**Decision**: Hydrate via a single `room:hydrate` event for canvases ≤50 KB of serialized stroke data; **chunk into sequential `room:hydrate:chunk` events** for larger payloads.

**Rationale**:
- FR-024 requires hydration within 1 000 ms for ≤500 strokes. A single JSON payload for 500 minimal strokes (`[{strokeId, operationId, userId, color, width, points:[{x,y}…]}…]`) is typically 30–80 KB — within a single-message limit.
- If serialized state exceeds a configurable threshold (default 50 KB), the server splits the stroke array into chunks of up to 100 strokes each, emitting them as:
  - `room:hydrate:start` `{ totalChunks, totalStrokes }`
  - `room:hydrate:chunk` `{ chunkIndex, strokes[] }` (repeated)
  - `room:hydrate:end` `{ roomCode }`
- The client buffers chunks and renders the committed canvas only after `room:hydrate:end`, preventing partial redraws.
- For MVP with ≤500 strokes, chunking will rarely trigger, but the code path must exist to satisfy FR-024's edge-case requirement.

**Alternatives considered**:
- *Delta compression (e.g., binary diff)*: More bandwidth efficient but requires a shared encoding library; deferred to post-MVP.
- *Paginated REST endpoint for initial state*: Avoids WebSocket message size limits but requires coordinating HTTP + WebSocket timing; adds latency.

---

## 5. Offline Stroke Queue & Reconnection

**Decision**: Buffer outbound `stroke:point` and `stroke:commit` events in a **client-side queue** (`Array`) during disconnection; flush in insertion order on reconnect, preserving original `operationId` values.

**Rationale**:
- SC-009 requires strokes drawn during a 5-second outage to appear within 3 s of reconnection.
- The `socket.js` service wraps emit calls. When `socket.connected` is `false`, events are pushed to `pendingQueue: Array<{event, payload}>` instead of being emitted.
- On the `connect` event (after backoff reconnection), the client emits `room:rejoin` first, waits for `room:hydrate` to complete (to avoid ordering issues), then flushes `pendingQueue` in order.
- `stroke:point` events in the queue are re-emitted as-is; the server renders them on existing in-progress strokes (identified by `strokeId`). Since the server only persists `stroke:commit`, any queued `stroke:point` events that arrive before their `stroke:commit` are treated as live incremental events.
- The server deduplicates `stroke:commit` by `operationId`, so retransmitting committed strokes is safe.

**Alternatives considered**:
- *Local Storage persistence*: Survives page refresh but adds serialization complexity; FR-010 / edge cases specify strokes are lost on refresh for MVP.
- *IndexedDB*: More robust but significantly more complex; deferred to post-MVP.

---

## 6. Exponential Backoff Reconnection

**Decision**: Implement backoff in `useConnection.js` using Socket.IO's built-in reconnection options, configured to match FR-020 requirements.

**Rationale**:
- Socket.IO Client 4.x has native reconnection support: `reconnection: true`, `reconnectionDelay`, `reconnectionDelayMax`, `randomizationFactor`.
- Configuration: `reconnectionDelay: 500`, `reconnectionDelayMax: 30000`, `randomizationFactor: 0.2` (±20 % jitter).
- `useConnection.js` listens to `connect`, `disconnect`, `reconnecting`, and `reconnect_failed` events to drive the `<ConnectionStatus>` component state machine (`connected → reconnecting → disconnected`).
- `reconnect_failed` fires after Socket.IO exhausts its attempts; at that point the hook sets status to `disconnected` and exposes a manual `retry()` function.

**Alternatives considered**:
- *Custom backoff implementation*: More control but duplicates work Socket.IO already provides correctly.

---

## 7. Mobile Touch Event Handling

**Decision**: Use the **Pointer Events API** (`pointerdown`, `pointermove`, `pointerup`, `pointercancel`) exclusively, which unifies mouse, touch, and stylus inputs.

**Rationale**:
- The Pointer Events API is supported in all target browsers and handles touch natively, eliminating the need for separate `touchstart`/`touchmove`/`touchend` handlers.
- `touch-action: none` is set on the canvas element via CSS to prevent browser default scroll/zoom behaviors during drawing.
- When no tool is active (future post-MVP) or on mobile viewport, the canvas should allow scroll — to be toggled by setting `touch-action: auto` on the container.
- `pointercancel` is treated as `pointerup` (stroke commit) to handle cases where the OS interrupts the gesture (e.g., notification).

**Alternatives considered**:
- *Separate Touch + Mouse handlers*: Redundant code; harder to maintain.

---

## 8. Page Visibility API — Drawing Suspension

**Decision**: Listen to `document.visibilitychange` in `useCanvas.js`; when the tab becomes hidden (`document.hidden === true`), suspend `pointermove` emit (but continue local rAF rendering so the canvas remains correct when the tab is foregrounded).

**Rationale**:
- FR specifies drawing input MUST be suspended when tab is hidden to avoid unnecessary payload.
- Suspending only the emit (not the local render loop) keeps the implementation simple and avoids a mid-stroke state flush.
- On visibility restore, normal emit resumes. Any in-progress stroke is committed when the pointer is lifted.

---

## Summary of Key Decisions

| Topic | Decision | Rationale |
|-------|----------|-----------|
| Canvas rendering | Dual-canvas architecture + rAF loop | Meets 60 fps budget; avoids React re-renders |
| stroke:point payload | Compact JSON, 2-decimal coords | ~100–110 bytes; within 128-byte limit |
| Conflict resolution | Server-ordered append-only log | Sufficient for atomic, independent strokes |
| State hydration | Single event < 50 KB; chunked ≥ 50 KB | Meets 1 000 ms hydration budget |
| Offline queue | In-memory Array flushed on reconnect | Meets SC-009; no persistent storage needed |
| Reconnection | Socket.IO built-in backoff (500 ms → 30 s, ±20 %) | Matches FR-020 exactly |
| Touch input | Pointer Events API only | Unified mouse/touch/stylus; all target browsers |
| Tab hidden | Suspend emit, continue rAF | Reduces payload; keeps local state consistent |
