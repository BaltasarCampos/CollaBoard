# Data Model: Whiteboard Collaboration MVP

**Branch**: `001-whiteboard-collab-mvp` | **Date**: 2026-05-10  
**Source**: [spec.md](spec.md) — Key Entities section + Functional Requirements

---

## Overview

All state lives **in server memory** for the MVP. There is no database. Server restart clears all rooms. The client holds a **derived, synchronized view** of server state; the server is the single source of truth.

---

## Entities

### Room

The top-level container for a collaborative session.

| Field | Type | Description |
|-------|------|-------------|
| `roomCode` | `string` | Unique, alphanumeric identifier (≥6 chars). Provided by the user or auto-generated. |
| `strokes` | `Map<strokeId, Stroke>` | Ordered map of all committed strokes. Insertion order is preserved and used for hydration replay. |
| `operationsProcessed` | `Set<operationId>` | Set of all `operationId` values that have been processed, used for server-side deduplication. |
| `participants` | `Map<userId, User>` | Currently connected users; keyed by `userId`. |
| `createdAt` | `number` | Unix timestamp (ms) when the room was created. |

**Lifecycle**:
- Created when the first user joins (whether the room code was provided or auto-generated).
- Destroyed **immediately** when the last participant disconnects (FR-010). No TTL-based cleanup for MVP.
- Room codes are validated to be alphanumeric; special characters are rejected (edge case from spec).

**Auto-generation**:
- Server generates a 6-character alphanumeric room code (case-insensitive; stored as uppercase).
- On collision, generation is retried up to 10 times (FR-013). If all 10 collide, server returns HTTP 500.

---

### Stroke

A finalized, durable drawing command. Only `stroke:commit` events produce `Stroke` records.

| Field | Type | Description |
|-------|------|-------------|
| `strokeId` | `string` | UUID v4. Identifies this stroke globally and is used as the target for erase/undo. |
| `operationId` | `string` | UUID v4. The idempotency key for the `stroke:commit` operation that created this stroke. |
| `userId` | `string` | UUID v4. The user who drew this stroke. Undo is restricted to same `userId` (FR-017). |
| `color` | `string` | CSS color string (e.g., `"#000000"`). Default: `"#000000"` (black). |
| `width` | `number` | Stroke width in logical pixels. Default: `3`. Range: `[1, 20]`. |
| `points` | `Array<Point>` | Ordered array of `{x, y}` coordinates forming the stroke path. Minimum 1 point. |
| `committedAt` | `number` | Unix timestamp (ms) when the server committed this stroke (server-assigned). |

**Notes**:
- `Stroke` objects are **immutable** after commit. Erasing removes the entry; undo also removes it (per-user, LIFO order).
- `points` coordinates are floating-point numbers rounded to 2 decimal places on the client before transmission.

---

### StrokePoint *(transit-only, never persisted)*

An incremental drawing delta emitted during an in-progress stroke. Used only in the `stroke:point` Socket.IO event; never stored on the server.

| Field | Type | Description |
|-------|------|-------------|
| `operationId` | `string` | UUID v4. Unique per entire stroke-in-progress session (same value for all points of one stroke). |
| `strokeId` | `string` | UUID v4. Links this point to the in-progress stroke it belongs to. |
| `x` | `number` | Canvas X coordinate (2 decimal places). |
| `y` | `number` | Canvas Y coordinate (2 decimal places). |

**Payload budget**: A `stroke:point` JSON payload with two 36-char UUIDs and two 6-char floats is approximately 105–115 bytes, within the 128-byte limit (FR-025).

---

### User

A connected socket session. Not persisted beyond the active session.

| Field | Type | Description |
|-------|------|-------------|
| `userId` | `string` | UUID v4. Generated server-side on first join. Sent back to the client and held in session memory. |
| `userName` | `string` | Display name provided at the landing page (non-empty, non-whitespace). Max 50 characters. |
| `socketId` | `string` | Socket.IO socket ID. Used to route per-user events and detect disconnect. |
| `roomCode` | `string` | The room this user is currently in. |
| `strokeHistory` | `Array<strokeId>` | Ordered list of `strokeId` values committed by this user in this session, used for per-user undo (LIFO). |
| `joinedAt` | `number` | Unix timestamp (ms) when the user joined the room. |

**Notes**:
- `userName` is validated client-side (non-empty, non-whitespace) and sanitized server-side (trimmed, max 50 chars) before use. Duplicate display names within a room are allowed; users are distinguished by `userId`.
- `strokeHistory` is cleared if the user refreshes (new session = new `userId`).

---

### Operation *(logical concept; not a stored entity)*

Any mutation of canvas state. `operationId` appears on every event payload and is the deduplication key. The server tracks processed `operationId` values per room in `Room.operationsProcessed`.

| Operation Type | Event | Idempotency |
|----------------|-------|-------------|
| `stroke:commit` | Client→Server→Broadcast | Server deduplicates by `operationId`; duplicate commits are silently dropped |
| `stroke:erase` | Client→Server→Broadcast | Deleting a non-existent `strokeId` is a no-op |
| `stroke:undo` | Client→Server→Broadcast | Undoing an already-removed `strokeId` is a no-op; `userId` check ensures only owner can undo |

---

### Point *(value object)*

A two-dimensional coordinate. Used within `Stroke.points`.

| Field | Type | Description |
|-------|------|-------------|
| `x` | `number` | Horizontal position in canvas logical pixels (2 decimal places). |
| `y` | `number` | Vertical position in canvas logical pixels (2 decimal places). |

---

## Client-Side State

The client maintains a **derived, in-memory view** of the collaborative state. It is not authoritative.

### Canvas State (held in `useCanvas` hook)

| Key | Type | Description |
|-----|------|-------------|
| `committedStrokes` | `Map<strokeId, Stroke>` | Finalized strokes received via `stroke:committed` or replayed from `room:hydrate`. |
| `remoteInProgress` | `Map<strokeId, Array<Point>>` | Incremental points from remote users' in-flight strokes (`stroke:point` events). Cleared when `stroke:committed` arrives for the same `strokeId`. |
| `localInProgress` | `{strokeId, operationId, points[]}` \| `null` | The current user's active stroke. `null` when not drawing. |
| `activeTool` | `'pen'` \| `'eraser'` | The currently selected tool. Defaults to `'pen'` on load. |

### Room State (held in `useRoom` hook)

| Key | Type | Description |
|-----|------|-------------|
| `roomCode` | `string` \| `null` | Current room code. `null` before joining. |
| `userId` | `string` \| `null` | This user's server-assigned UUID. |
| `userName` | `string` | Display name submitted at the landing page. |
| `participants` | `Array<{userId, userName}>` | Current room participants. |
| `undoStack` | `Array<strokeId>` | This user's undoable strokes in commit order (LIFO undo). Mirrors `User.strokeHistory` server-side. |
| `pendingQueue` | `Array<{event, payload}>` | Events buffered during disconnection, flushed on reconnect. |

### Connection State (held in `useConnection` hook)

| Key | Type | Description |
|-----|------|-------------|
| `status` | `'connected'` \| `'reconnecting'` \| `'disconnected'` | Drives `<ConnectionStatus>` component. |
| `retryCount` | `number` | Number of reconnection attempts made in the current outage window. |

---

## State Transitions

### Room Lifecycle

```
[No room] 
  → user submits form → Server: room created or joined
  → [Room active: participants ≥ 1]
  → last participant disconnects → Server: room destroyed
  → [No room]
```

### Stroke Lifecycle

```
[No stroke]
  → pointerdown → localInProgress created (strokeId generated)
  → pointermove → stroke:point emitted + localInProgress.points appended
  → pointerup → stroke:commit emitted → localInProgress cleared
[stroke:committed received / own stroke acknowledged]
  → committedStrokes.set(strokeId, stroke) + undoStack.push(strokeId)
[stroke:erase or stroke:undo received]
  → committedStrokes.delete(strokeId) + undoStack.remove(strokeId) [if own]
```

### Connection State Machine

```
connected
  → socket disconnect → reconnecting (backoff starts: 500 ms → 30 s, ±20%)
  → reconnect success → room:rejoin emitted → room:hydrate received → connected
  → reconnect failure (> 30 s) → disconnected
disconnected
  → user clicks Retry → reconnecting
```

---

## Validation Rules

| Entity | Field | Rule |
|--------|-------|------|
| User | `userName` | Non-empty after trim; ≤50 characters; validated client-side before submit and sanitized server-side |
| Room | `roomCode` (provided) | Alphanumeric only (`/^[A-Z0-9]+$/i`); 1–20 characters; sanitized server-side |
| Room | `roomCode` (auto-generated) | 6-character uppercase alphanumeric; generated server-side |
| Stroke | `color` | Valid CSS color string; defaults to `"#000000"` if absent/invalid |
| Stroke | `width` | Integer in `[1, 20]`; defaults to `3` if absent/invalid |
| Stroke | `points` | Array with ≥1 `{x, y}` element; floats, 2 decimal places |
| StrokePoint | `x`, `y` | Finite floats; rejected if NaN or ±Infinity |
| Operation | `operationId` | UUID v4 format; rejected if malformed |
| Operation | `strokeId` | UUID v4 format; rejected if malformed |
