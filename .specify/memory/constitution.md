<!--
  SYNC IMPACT REPORT
  Version Change: [TEMPLATE] → 1.0.0
  Modified Principles: N/A — initial constitution population
  Added Sections:
    - Core Principles I–VIII (Test-First, Modularity, Event-Driven Architecture,
      Idempotency, Convergence & Conflict Resolution, Loose Coupling, Observability,
      Resilience)
    - Technology Stack
    - Development Workflow
    - Governance
  Removed Sections: N/A
  Templates Updated:
    ✅ .specify/memory/constitution.md (this file)
    ✅ .specify/templates/plan-template.md — Constitution Check gates updated
    ✅ .specify/templates/spec-template.md — No structural changes required
    ✅ .specify/templates/tasks-template.md — No structural changes required
  Follow-up TODOs: None
-->

# Colla-Board Constitution

## Core Principles

### I. Test-First (NON-NEGOTIABLE)

TDD is mandatory for all new features and bug fixes. Tests MUST be written
and user-approved before any implementation begins. The cycle MUST follow:
User Approved → Red (failing test) → Green (passing) → Refactor.

- Unit tests cover individual hooks, services, and utilities in isolation.
- Integration tests cover Socket.IO event flows and client–server contracts.
- E2E tests cover full collaborative sessions (multi-user draw, undo/redo, sync).
- No code MUST be merged if it causes a previously passing test to fail
  without explicit justification.

### II. Modularity

Whiteboard features MUST be implemented as independent, self-contained modules.

- Each tool (pen, shapes, text, cursors, chat) MUST live in its own module
  with a clear, single responsibility.
- Modules MUST be independently testable without requiring other tool modules
  to be loaded.
- Cross-module communication MUST go through defined interfaces (events or
  service contracts), never direct internal calls.
- Adding or removing a tool MUST NOT require changes to unrelated modules.

### III. Event-Driven Architecture

All real-time collaboration MUST be built on an event-driven model.

- Every user action (draw stroke, move element, undo, redo, cursor move) MUST
  be represented as a discrete, named event.
- The server MUST act as an event broker via Socket.IO — it routes events to
  room participants without owning rendering logic.
- Client components MUST react to incoming events; they MUST NOT poll for state.
- Event payloads MUST be versioned and documented (name, schema, direction:
  client→server or server→client).

### IV. Idempotency

All whiteboard operations MUST be idempotent.

- Applying the same operation (add shape, move element, delete stroke) multiple
  times MUST produce the same final state as applying it once.
- Every operation MUST carry a unique `operationId` (UUID) so duplicates can be
  detected and discarded.
- The server MUST deduplicate operations before broadcasting to room participants.
- Network retries MUST be safe — re-emitting an event MUST NOT corrupt shared
  canvas state.

### V. Convergence & Conflict Resolution

Concurrent edits from multiple users MUST converge to a consistent shared state.

- Conflict resolution MUST be deterministic — given the same set of operations,
  all clients MUST arrive at the same canvas state regardless of arrival order.
- Operations MUST be ordered by a logical timestamp (server-assigned sequence
  number or vector clock).
- The server MUST be the authority for operation ordering within a room.
- Data loss during concurrent edits is NEVER acceptable; conflicting operations
  MUST be merged, not discarded.
- The resolution strategy (last-writer-wins per element, or OT/CRDT approach)
  MUST be documented and consistently applied.

### VI. Loose Coupling

Components and services MUST minimize direct dependencies.

- Client components MUST communicate with the server exclusively via the socket
  service (`client/src/services/socket.js`); no component MUST import server
  modules directly.
- Server-side room logic (`roomService.js`) MUST NOT depend on specific client
  event names — event binding belongs in `eventHandlers.js`.
- Shared data schemas (event payloads, room state) MUST be defined in
  `constants.js` files and imported, never duplicated inline.
- Replacing or upgrading any single layer (e.g., swapping Socket.IO for another
  transport) MUST require changes only in the adapter layer, not throughout the
  codebase.

### VII. Observability

The system MUST expose sufficient data to understand behavior in production.

- The server MUST emit structured JSON logs (via `server/utils/logger.js`) for
  every socket connection, disconnection, room join/leave, and operation broadcast.
- Log entries MUST include: `timestamp`, `level`, `roomId`, `userId`, `event`,
  and `durationMs` where applicable.
- Critical errors (unhandled socket errors, room state corruption) MUST be logged
  at ERROR level with full context.
- Client-side errors (failed syncs, dropped connections) MUST be surfaced in the
  UI and logged to the browser console in development.
- Performance-impacting paths (large canvas state hydration, bulk undo replay)
  MUST be instrumented with timing metrics.

### VIII. Resilience

The system MUST degrade gracefully under failure conditions.

- A client that loses its WebSocket connection MUST automatically attempt
  reconnection with exponential backoff.
- On reconnection, the client MUST re-hydrate canvas state from the server —
  local state MUST NOT be assumed to be current after a disconnect.
- If the server cannot reach a room's state (e.g., memory eviction), it MUST
  respond with a recoverable error, not a crash.
- The UI MUST inform users of connection status (connected / reconnecting /
  disconnected) at all times.
- Room state persistence strategy (in-memory with TTL, or backed storage) MUST
  be documented and tested for recovery scenarios.

## Technology Stack

- **Frontend**: React, Canvas API, Socket.IO client
- **Backend**: Node.js, Express, Socket.IO server
- **Testing**: Jest, React Testing Library (unit + integration), Playwright or
  similar (E2E)
- **Monorepo**: `client/` for frontend, `server/` for backend; shared constants
  via respective `utils/constants.js` files
- All stack changes MUST be evaluated against Principles I–VIII before adoption.

## Development Workflow

- All work MUST begin with a failing test (Principle I).
- Feature branches MUST reference a spec or task from `.specify/`.
- PRs MUST include: passing tests, updated docs if contracts changed, and a
  Principle compliance note.
- Socket.IO event contracts (name + payload schema) MUST be documented whenever
  added or changed.
- The server MUST be the single source of truth for room state; client-side
  state is a derived, synchronized view.

## Governance

This constitution supersedes all other practices for the Colla-Board project.
Amendments require documentation, explicit approval, and a migration plan when
existing contracts or module boundaries change.

- All PRs and reviews MUST verify compliance with the principles above.
- Any deliberate deviation from a principle MUST be justified in writing and
  approved before merging.
- Amendments increment the version per semantic versioning: MAJOR for
  backward-incompatible governance changes, MINOR for new principles or material
  expansions, PATCH for clarifications and wording refinements.
- `LAST_AMENDED_DATE` MUST be updated on every amendment to the current date.

**Version**: 1.0.0 | **Ratified**: 2026-05-09 | **Last Amended**: 2026-05-09
