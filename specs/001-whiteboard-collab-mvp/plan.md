# Implementation Plan: Whiteboard Collaboration MVP

**Branch**: `001-whiteboard-collab-mvp` | **Date**: 2026-05-10 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from `/specs/001-whiteboard-collab-mvp/spec.md`

---

## Summary

Build a real-time collaborative whiteboard MVP with a React (Canvas API) frontend and a Node.js + Express + Socket.IO backend. Users create or join rooms by display name and optional room code, draw freehand strokes that are broadcast **point-by-point** to all room participants, and can erase or undo their own strokes. The system targets 60 fps local rendering, sub-150 ms remote segment delivery, and a ≤250 KB gzipped initial bundle.

---

## Technical Context

**Language/Version**: JavaScript — Node.js LTS 22 (server), ES2022 (client transpiled by Vite)  
**Primary Dependencies**:
- *Client*: React 18, Socket.IO Client 4.x, Vite 5.x (build/dev server)
- *Server*: Express 4.x, Socket.IO Server 4.x

**Storage**: In-memory (`Map` objects on the server process); no database for MVP  
**Testing**: Jest + React Testing Library (unit + integration), Playwright (E2E)  
**Target Platform**: Evergreen browsers (Chrome, Firefox, Edge, Safari — latest 2 versions); responsive 375 px–1 920 px  
**Project Type**: Web application monorepo — `client/` (React SPA) + `server/` (Node.js/Express + Socket.IO)  
**Performance Goals**: ≤16 ms/frame local stroke rendering, ≤150 ms P99 remote segment delivery, ≤1 000 ms room hydration for ≤500 committed strokes
**Constraints**: ≤128 bytes per `stroke:point` payload, ≤250 KB gzipped client bundle, ≤3 s TTI on simulated 4G, no persistence beyond active session; single-server; no auth;
**Scale/Scope**: Single Node.js process; MVP targets small-to-medium rooms (tens of concurrent users per room)

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design.*

| Principle | Gate Question | Status |
|-----------|--------------|--------|
| I. Test-First | Are failing tests written and approved before any implementation begins? | ✅ Unit/integration/E2E test files are scaffolded and must pass (fail-first) before corresponding source files are written |
| II. Modularity | Is the feature scoped to a single module with clear responsibility? Does it avoid changes to unrelated modules? | ✅ Canvas, Toolbar, ParticipantsPanel, and ConnectionStatus are independent components; Pen and Eraser are independent tool strategies |
| III. Event-Driven | Are all user actions represented as discrete, named Socket.IO events? Are payloads documented? | ✅ All mutations (stroke:point, stroke:commit, stroke:erase, stroke:undo, room:join, room:rejoin) are named events with documented schemas in `contracts/socket-events.md` |
| IV. Idempotency | Does every operation carry a unique `operationId`? Is server-side deduplication addressed? | ✅ Every mutation carries a UUID v4 `operationId`; server deduplicates by `operationId` before persisting and broadcasting |
| V. Convergence | Is the conflict-resolution strategy documented and deterministic? | ✅ Server-ordered append-only stroke log; idempotent erase by `strokeId`; last-receipt-wins for concurrent erases of the same stroke; documented in `research.md` |
| VI. Loose Coupling | Does client code communicate only via `socket.js`? Are shared schemas in `constants.js`? | ✅ All socket calls go through `client/src/services/socket.js`; event names and payload shapes defined in `client/src/utils/constants.js` and `server/utils/constants.js` |
| VII. Observability | Are structured log points specified for connections, events, and errors? Are timing metrics planned for perf paths? | ✅ `server/utils/logger.js` emits structured JSON for join/leave/create/destroy/unhandled errors; hydration path instrumented with `durationMs` |
| VIII. Resilience | Is reconnection and state re-hydration behavior defined? Is connection status surfaced in the UI? | ✅ Exponential backoff (500 ms initial, 30 s max, ±20 % jitter); `room:rejoin` on reconnect; `<ConnectionStatus>` component always visible; offline stroke queue flushed on reconnect |

All gates pass. No violations require justification.

---

## Project Structure

### Documentation (this feature)

```text
specs/001-whiteboard-collab-mvp/
├── plan.md              # This file (/speckit.plan output)
├── research.md          # Phase 0 output (/speckit.plan)
├── data-model.md        # Phase 1 output (/speckit.plan)
├── quickstart.md        # Phase 1 output (/speckit.plan)
├── contracts/           # Phase 1 output (/speckit.plan)
│   └── socket-events.md
└── tasks.md             # Phase 2 output (/speckit.tasks — NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
client/
├── src/
│   ├── components/
│   │   ├── LandingPage/
│   │   │   ├── LandingPage.jsx         # Username + room code form
│   │   │   └── LandingPage.test.jsx
│   │   ├── WhiteboardRoom/
│   │   │   ├── WhiteboardRoom.jsx      # Root room view: canvas + toolbar + panels
│   │   │   └── WhiteboardRoom.test.jsx
│   │   ├── Canvas/
│   │   │   ├── Canvas.jsx              # <canvas> element, rAF render loop
│   │   │   └── Canvas.test.jsx
│   │   ├── Toolbar/
│   │   │   ├── Toolbar.jsx             # Pen / Eraser / Undo layout
│   │   │   ├── ToolButton.jsx          # Reusable FR-029 component (sole button implementation)
│   │   │   └── Toolbar.test.jsx
│   │   ├── ParticipantsPanel/
│   │   │   ├── ParticipantsPanel.jsx   # Live participant list
│   │   │   └── ParticipantsPanel.test.jsx
│   │   └── ConnectionStatus/
│   │       ├── ConnectionStatus.jsx    # Connected / Reconnecting / Disconnected badge
│   │       └── ConnectionStatus.test.jsx
│   ├── hooks/
│   │   ├── useCanvas.js                # Drawing state, local rAF loop
│   │   ├── useDrawing.js
│   │   ├── useRoom.js                  # Room join/leave, hydration, participant list
│   │   └── useConnection.js            # Socket status + backoff timers
│   ├── services/
│   │   ├── canvasService.js
│   │   ├── drawingEngine.js
│   │   └── socket.js                   # Singleton Socket.IO client; only file that imports socket.io-client
│   └── utils/
│       └── constants.js                # EVENT_NAMES, payload schemas, config (bundle budget, payload limits)
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/                            # Playwright multi-tab collaborative sessions
└── vite.config.js

server/
├── src/
│   ├── server.js                       # Express + Socket.IO bootstrap, HTTP server
│   ├── eventHandlers.js                # Registers socket event listeners; NO business logic
│   └── roomService.js                  # Pure room-state mutations (create, join, stroke ops, undo)
└── utils/
    ├── constants.js                    # Event names + payload schemas (mirrors client constants)
    └── logger.js                       # Structured JSON logger (wraps console; upgradeable to pino)
```

**Structure Decision**: Web application (Option 2) — `client/` and `server/` top-level directories in a lightweight monorepo. Event name strings and payload shapes are defined separately in each package's `utils/constants.js` to avoid a shared-package build step for the MVP. Upgrading to a shared `packages/contracts` package is a natural post-MVP step.

---

## Complexity Tracking

No Constitution violations require justification for this feature.

**FR-032 logging scope (Constitution §VII)**: Per-event broadcast logging
(`stroke:point`, `stroke:commit`, etc.) is intentionally excluded from FR-032
to avoid high-volume log noise at scale. The post-MVP upgrade path is a
structured log sampler or dedicated telemetry layer. This is a documented
trade-off, not a Constitution violation.
