---
description: "Task list for Whiteboard Collaboration MVP implementation"
---

# Tasks: Whiteboard Collaboration MVP

**Input**: Design documents from `/specs/001-whiteboard-collab-mvp/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/socket-events.md ✅, quickstart.md ✅

**Tests**: Included — Test-First is a passing Constitution gate (plan.md §Constitution Check, Principle I). All test tasks MUST be written and confirmed failing before the corresponding implementation tasks are started.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

---

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US4, US3, US5)
- Exact file paths are included in every task description

## Path Conventions

- **Web app monorepo**: `client/` (React SPA) and `server/` (Node.js/Express + Socket.IO) at repository root
- Client source: `client/src/`; Client tests: `client/tests/` + co-located `*.test.jsx` files
- Server source: `server/src/`; Server utilities: `server/utils/`; Server tests: `server/tests/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Initialize the monorepo, install dependencies, and configure all tooling. No story-specific logic yet.

- [ ] T001 Initialize monorepo root with `package.json` (npm workspaces for `client` and `server`), `.nvmrc` (Node 22), and root `.gitignore`
- [ ] T002 Initialize `server/package.json` with Express 4.x, Socket.IO Server 4.x, uuid, dotenv as dependencies and Jest, nodemon as devDependencies; add `dev`, `start`, `test`, and `test:watch` npm scripts
- [ ] T003 [P] Initialize `client/package.json` with React 18, Vite 5.x, Socket.IO Client 4.x as dependencies and Jest, @testing-library/react, @testing-library/jest-dom, @testing-library/user-event, @vitejs/plugin-react, Playwright as devDependencies; add `dev`, `build`, `preview`, `test`, `test:watch`, and `test:e2e` npm scripts
- [ ] T004 [P] Configure Vite with `@vitejs/plugin-react`, dev-server proxy (`/socket.io` → `http://localhost:3001`), and `build.chunkSizeWarningLimit: 250` in `client/vite.config.js`
- [ ] T005 [P] Configure Jest with Node test environment, coverage (`lcov`, `text`), and `testMatch: ['**/tests/unit/**/*.test.js']` in `server/jest.config.js`
- [ ] T006 [P] Configure Jest with jsdom test environment, `setupFilesAfterFramework: ['@testing-library/jest-dom/extend-expect']`, and module name mapper for static assets in `client/jest.config.js`
- [ ] T007 Configure Playwright with `baseURL: 'http://localhost:5173'`, `retries: 1`, headless mode, and a `chromium` project in `client/playwright.config.js`

**Checkpoint**: `npm install` succeeds for both packages; `npm test` scaffolding runs (no test files yet); `npm run dev` skeleton commands are registered.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core server infrastructure — constants, logger, room state management, Socket.IO event wiring, and the client socket singleton. This phase MUST complete before any user story work can begin.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T008 [P] Define `EVENT_NAMES` (all Socket.IO event name strings) and `ERROR_CODES` (`ROOM_NOT_FOUND`, `ROOM_GENERATION_FAILED`, `INVALID_USER`, `STROKE_NOT_FOUND`, `STROKE_NOT_OWNED`, `VALIDATION_ERROR`) as frozen constants in `server/utils/constants.js`
- [ ] T009 [P] Define `EVENT_NAMES` (mirrors server), `PAYLOAD_MAX_BYTES: 128`, `HYDRATE_CHUNK_THRESHOLD_BYTES: 51200`, and `round2(n)` coordinate-truncation helper (`Math.round(n * 100) / 100`) in `client/src/utils/constants.js`
- [ ] T010 [P] Implement structured JSON logger that emits `{ timestamp, eventType, ...identifiers }` to stdout using `console.log`; export `logger.info`, `logger.warn`, `logger.error` in `server/utils/logger.js`
- [ ] T011 Write unit tests for `roomService` covering: `createRoom` generates 6-char alphanumeric code and retries on collision (up to 10), `joinRoom` adds user to existing or new room, `addStroke` persists and deduplicates by `operationId`, `eraseStroke` removes by `strokeId` (no-op if absent), `undoStroke` removes LIFO from `user.strokeHistory` and validates ownership, `destroyRoom` removes room when last user leaves; in `server/tests/unit/roomService.test.js`
- [ ] T012 Implement `roomService.js` with in-memory `Map`-based `Room`/`User` state: `createRoom`, `joinRoom`, `leaveRoom`, `addStroke`, `eraseStroke`, `undoStroke`, `hydrateRoom` (returns ordered strokes for chunking), and `destroyRoom`; all pure functions returning updated state or throwing on validation failure; in `server/src/roomService.js`
- [ ] T013 Write unit tests for `eventHandlers` verifying that each Socket.IO event (`room:join`, `room:rejoin`, `stroke:point`, `stroke:commit`, `stroke:erase`, `stroke:undo`, `disconnect`) delegates to the corresponding `roomService` function with correct arguments; use mock `socket` and `io` objects; in `server/tests/unit/eventHandlers.test.js`
- [ ] T014 Implement `eventHandlers.js` that registers all Socket.IO event listeners on a given `socket`, calls `roomService` methods, broadcasts results via `io.to(roomCode).emit(...)`, and emits `error` events for validation failures; no business logic in this file; in `server/src/eventHandlers.js`
- [ ] T015 Bootstrap Express app and Socket.IO server: load `.env` (`PORT`, `CLIENT_ORIGIN`, `HYDRATE_CHUNK_THRESHOLD_BYTES`), configure CORS for `CLIENT_ORIGIN`, attach Socket.IO to the HTTP server, register `eventHandlers` for each connected socket, and start listening; in `server/src/server.js`
- [ ] T016 Implement singleton Socket.IO client: connect to `VITE_SERVER_URL` (env var), export typed `emit`, `on`, `off`, and `once` wrappers; expose a `connected` getter; do NOT auto-connect — call `socket.connect()` explicitly on landing page submit; in `client/src/services/socket.js`

**Checkpoint**: Server starts with `npm run dev`; Socket.IO handshake succeeds from a browser console `io('http://localhost:3001')` call; all unit tests in Phase 2 pass.

---

## Phase 3: User Story 1 — Room Creation and Joining (Priority: P1) 🎯 MVP

**Goal**: A user can enter a display name and optional room code on the landing page, create or join a room, and see the whiteboard shell. A second user entering the same room code joins the same session and receives the committed canvas state.

**Independent Test**: Open the app in two fresh browser tabs. Tab A enters a name and submits (no room code) → WhiteboardRoom shell renders with a room code shown in the header. Tab B enters the same room code → both appear in the participants list; the canvas state from Tab A is replayed to Tab B.

### Tests for User Story 1

> **Write these tests FIRST and confirm they FAIL before implementing the components below.**

- [ ] T017 [P] [US1] Write unit tests for `LandingPage`: non-empty username required, username trimmed whitespace blocked, room code field is optional, successful form submission emits `room:join` via socket service, error displayed when socket emits `error` event; in `client/src/components/LandingPage/LandingPage.test.jsx`
- [ ] T018 [P] [US1] Write integration tests for the room join + hydration cycle: mock socket broadcasts `room:joined` then `room:hydrate` → `useRoom` state contains correct `roomCode`, `userId`, `participants`, and `committedStrokes`; in `client/tests/integration/roomFlow.test.js`

### Implementation for User Story 1

- [ ] T019 [P] [US1] Implement `LandingPage` component: controlled form with `userName` (required) and `roomCode` (optional) fields, client-side validation (non-empty after trim), inline error message for empty username, responsive layout (375 px–1 920 px), emits `room:join` on submit; in `client/src/components/LandingPage/LandingPage.jsx`
- [ ] T020 [P] [US1] Implement `useRoom` hook managing state: `{ roomCode, userId, userName, participants, undoStack, pendingQueue, isJoined }`; handle `room:joined`, `room:hydrate`, `room:hydrate:start`, `room:hydrate:chunk`, `room:hydrate:end`, `room:participant_joined`, and `room:participant_left` socket events; expose `joinRoom(userName, roomCode)` and `leaveRoom()` actions; in `client/src/hooks/useRoom.js`
- [ ] T021 [US1] Implement `WhiteboardRoom` shader component with layout sections: `<canvas>` placeholder (`data-testid="canvas-placeholder"`), toolbar placeholder, participants column placeholder, and connection status placeholder; receives `roomData` prop from `useRoom`; shows room code in header; in `client/src/components/WhiteboardRoom/WhiteboardRoom.jsx`
- [ ] T022 [US1] Create client entry point and view switching: `client/src/main.jsx` (mounts `<App />`); `client/src/App.jsx` renders `<LandingPage>` when not joined or `<WhiteboardRoom>` when joined, driven by `useRoom().isJoined`

**Checkpoint**: User Story 1 is fully functional and independently testable. Two tabs can join the same room; hydrated canvas state is replayed on join.

---

## Phase 4: User Story 2 — Real-Time Freehand Drawing (Priority: P1)

**Goal**: A user draws freehand strokes that appear locally at ≤16 ms/frame and are visible to all room participants point-by-point within 150 ms (P99). Committed strokes are stored idempotently. Erasing and undoing a stroke synchronises to all users.

**Independent Test**: Open two browser tabs in the same room. Draw a continuous line in Tab A. Tab B must render each incremental segment as the pointer moves — not after pointer-up. Erase the stroke in Tab A; it disappears in Tab B. Ctrl+Z in Tab A removes the last committed stroke from both tabs.

### Tests for User Story 2

> **Write these tests FIRST and confirm they FAIL before implementing the services/hooks/components below.**

- [ ] T023 [P] [US2] Write unit tests for `drawingEngine`: `renderFrame` draws committed strokes on the committed canvas layer and in-progress strokes on the in-progress layer; clearing is isolated per layer; `drawStroke` correctly calls `ctx.moveTo`/`ctx.lineTo`/`ctx.stroke`; in `client/tests/unit/drawingEngine.test.js`
- [ ] T024 [P] [US2] Write unit tests for `useCanvas` hook: `localInProgress` is created on pointerdown and cleared on pointerup, `committedStrokes` map updates on `stroke:committed` and removes on `stroke:erased`/`stroke:undone`, `remoteInProgress` accumulates points keyed by `strokeId`; in `client/tests/unit/useCanvas.test.js`
- [ ] T025 [P] [US2] Write unit tests for `Canvas` component: renders two `<canvas>` elements (committed layer + in-progress layer), `pointerdown` creates a new local stroke, `pointermove` appends a point and emits `stroke:point`, `pointerup` emits `stroke:commit`; in `client/src/components/Canvas/Canvas.test.jsx`

### Implementation for User Story 2

- [ ] T026 [P] [US2] Implement `drawingEngine.js` with dual-canvas rendering strategy (per `research.md §1`): `startRenderLoop(committedCanvasRef, inProgressCanvasRef, getState)` drives a `requestAnimationFrame` loop; `drawStroke(ctx, stroke)` renders a single stroke; `clearInProgressLayer(ctx)` clears the in-progress canvas; apply `round2` coordinate truncation before rendering; in `client/src/services/drawingEngine.js`
- [ ] T027 [P] [US2] Implement `canvasService.js` managing stroke data: `addCommittedStroke(stroke)`, `removeStroke(strokeId)`, `getStrokesNearPoint(x, y, tolerance)` (returns array of `strokeId`s whose paths pass within `tolerance` px of the given point — used by eraser hit detection); in `client/src/services/canvasService.js`
- [ ] T028 [US2] Implement `useCanvas` hook managing: `localInProgress: { strokeId, operationId, points[] } | null`, `committedStrokes: Map<strokeId, Stroke>`, `remoteInProgress: Map<strokeId, Point[]>`, `activeTool: 'pen' | 'eraser'`; expose `setActiveTool`, `startLocalStroke`, `appendLocalPoint`, `commitLocalStroke`, `applyRemotePoint`, `applyCommittedStroke`, `applyErase`, `applyUndo`; in `client/src/hooks/useCanvas.js`
- [ ] T029 [US2] Implement `useDrawing` hook: bind `pointerdown`, `pointermove`, `pointerup`, `touchstart`, `touchmove`, `touchend` on the canvas ref; on `pointermove` (pen mode) call `round2` on coordinates and emit `stroke:point`; on `pointerup` emit `stroke:commit` with full points array, `color`, and `width`; pause drawing when `document.visibilityState === 'hidden'` (Page Visibility API); in `client/src/hooks/useDrawing.js`
- [ ] T030 [US2] Implement `Canvas` component: render `<div>` containing two stacked `<canvas>` elements (committed layer below, in-progress layer above); forward both refs to `drawingEngine.startRenderLoop`; delegate pointer/touch events to `useDrawing`; fill viewport minus toolbar height; in `client/src/components/Canvas/Canvas.jsx`
- [ ] T031 [US2] Handle incoming Socket.IO drawing events in `useCanvas`: `stroke:point` → `applyRemotePoint`; `stroke:committed` → `applyCommittedStroke` + push to `undoStack`; `stroke:erased` → `applyErase` (`removeStroke`); `stroke:undone` → `applyUndo` (`removeStroke`); wire these handlers in `socket.js` event subscriptions called from `useCanvas`; integrate updated `Canvas` into `WhiteboardRoom`
- [ ] T032 [US2] Implement offline stroke queue in `useRoom.js`: when `socket.connected === false`, push `{ event, payload }` to `pendingQueue` instead of emitting; do NOT flush queue yet (flush implemented in US4 Phase 5); in `client/src/hooks/useRoom.js`

**Checkpoint**: User Story 2 is fully functional and independently testable. Two tabs collaborate on drawing. Closing the keyboard shortcut for undo is wired. Erase syncs across tabs.

---

## Phase 5: User Story 4 — Connection Status Feedback (Priority: P2)

**Goal**: The UI always displays the current connection state (connected / reconnecting / disconnected). On disconnect the client begins exponential backoff, re-hydrates canvas state on reconnect, and flushes any offline-queued strokes.

**Independent Test**: Disable the network while in a room → amber "Reconnecting…" badge appears within 500 ms. Re-enable → badge turns "Connected", canvas re-hydrates, and any strokes drawn offline appear for all users within 3 s.

### Tests for User Story 4

> **Write these tests FIRST and confirm they FAIL before implementing the hook and component below.**

- [ ] T033 [P] [US4] Write unit tests for `useConnection` hook: status transitions (`connected → reconnecting` on disconnect, `reconnecting → connected` on reconnect, `reconnecting → disconnected` after >30 s), `retryCount` increments on each attempt, backoff delay doubles with ±20 % jitter (assert within range); in `client/tests/unit/useConnection.test.js`
- [ ] T034 [P] [US4] Write unit tests for `ConnectionStatus` component: renders "Connected" badge (green) when `status='connected'`, "Reconnecting…" badge (amber) when `status='reconnecting'`, "Disconnected" badge (red) + Retry button when `status='disconnected'`; Retry button calls `onRetry` prop; in `client/src/components/ConnectionStatus/ConnectionStatus.test.jsx`

### Implementation for User Story 4

- [ ] T035 [P] [US4] Implement `useConnection` hook: subscribe to socket `connect`, `disconnect`, and `reconnect_attempt` events; derive `status` and `retryCount`; configure Socket.IO client with `reconnectionDelay: 500`, `reconnectionDelayMax: 30000`, `randomizationFactor: 0.2`; expose `reconnect()` for the manual Retry action; in `client/src/hooks/useConnection.js`
- [ ] T036 [US4] Implement `ConnectionStatus` component: non-intrusive fixed badge (top-right corner, `role="status"`, `aria-live="polite"`); three visual states driven by `status` prop; "Retry" button visible only on `disconnected` state; uses `role` and `aria-label` for WCAG 2.1 AA compliance; in `client/src/components/ConnectionStatus/ConnectionStatus.jsx`
- [ ] T037 [US4] Implement reconnection flow in `useRoom.js`: on socket `connect` event (after backoff), emit `room:rejoin { userId, roomCode }`, wait for `room:hydrate` (or chunked sequence) to complete, then flush `pendingQueue` in insertion order (re-emit each `{ event, payload }`); clear `pendingQueue` after flush; in `client/src/hooks/useRoom.js`
- [ ] T038 [US4] Integrate `ConnectionStatus` into `WhiteboardRoom`: replace connection status placeholder with `<ConnectionStatus status={connectionStatus} onRetry={reconnect} />`; wire `useConnection` status and `reconnect` action; in `client/src/components/WhiteboardRoom/WhiteboardRoom.jsx`

**Checkpoint**: User Story 4 is fully functional and independently testable. Connection badge reflects real state. Offline strokes flush on reconnect.

---

## Phase 6: User Story 3 — Participants Panel (Priority: P3)

**Goal**: The whiteboard view shows a live list of all connected users. The list updates in real time as users join or leave, without a page refresh.

**Independent Test**: Open two tabs in the same room — both names appear in the panel. Close Tab B — Tab A's panel removes that name within 5 s.

### Tests for User Story 3

> **Write these tests FIRST and confirm they FAIL before implementing the component below.**

- [ ] T039 [P] [US3] Write unit tests for `ParticipantsPanel` component: renders name for each participant in the `participants` prop, includes the current user's own name, updates when the `participants` array changes (simulate join/leave), shows participant count; in `client/src/components/ParticipantsPanel/ParticipantsPanel.test.jsx`

### Implementation for User Story 3

- [ ] T040 [US3] Implement `ParticipantsPanel` component: displays a sorted list of `{ userId, userName }` entries from the `participants` prop; highlights the current user (`userId` match); accessible list with `aria-label="Participants"`; responsive column layout; in `client/src/components/ParticipantsPanel/ParticipantsPanel.jsx`
- [ ] T041 [US3] Integrate `ParticipantsPanel` into `WhiteboardRoom`: replace participants placeholder with `<ParticipantsPanel participants={participants} currentUserId={userId} />`; wire from `useRoom`; in `client/src/components/WhiteboardRoom/WhiteboardRoom.jsx`

**Checkpoint**: User Story 3 is fully functional and independently testable. Two users see each other's names. Closing a tab removes the name in the other tab within 5 s.

---

## Phase 7: User Story 5 — Tool Selection: Pen and Eraser (Priority: P3)

**Goal**: The user can switch between Pen (default) and Eraser tools from a consistently placed toolbar. The active tool is visually highlighted. Undo removes the user's last committed stroke. All toolbar buttons use the single `ToolButton` component.

**Independent Test**: Draw a stroke. Switch to Eraser. Drag over the stroke → it disappears for all users. Switch back to Pen. Ctrl+Z → last committed stroke undoes for all users. Undo button is disabled when there are no strokes left.

### Tests for User Story 5

> **Write these tests FIRST and confirm they FAIL before implementing the components below.**

- [ ] T042 [P] [US5] Write unit tests for `ToolButton` component: renders with correct `aria-label`, applies active CSS class when `isActive={true}`, calls `onClick` on click, is keyboard-accessible; confirm this is the sole toolbar button implementation (no other buttons in Toolbar); in `client/src/components/Toolbar/Toolbar.test.jsx`
- [ ] T043 [P] [US5] Write unit tests for `canvasService.getStrokesNearPoint`: returns `strokeId`s whose path segments pass within the given `tolerance` px radius; returns empty array when no strokes overlap; handles strokes with a single point; in `client/tests/unit/canvasService.test.js`

### Implementation for User Story 5

- [ ] T044 [P] [US5] Implement `ToolButton` component: single reusable button accepting `label`, `icon`, `isActive`, `disabled`, `onClick`, and `aria-label` props; active state applies a distinct highlighted style; meets WCAG 2.1 AA contrast; this MUST be the only button implementation used in the toolbar (SC-010); in `client/src/components/Toolbar/ToolButton.jsx`
- [ ] T045 [US5] Implement `Toolbar` component: renders three `ToolButton`s — Pen (default active), Eraser, and Undo; Undo `ToolButton` is `disabled` when `undoStack.length === 0`; exposes `onToolChange(tool)` and `onUndo()` props; consistent horizontal layout with clear active-state indication; in `client/src/components/Toolbar/Toolbar.jsx`
- [ ] T046 [US5] Add eraser mode to `useDrawing`: when `activeTool === 'eraser'`, `pointerdown`/`pointermove` call `canvasService.getStrokesNearPoint(x, y, tolerancePx)` and emit `stroke:erase { operationId, targetStrokeId }` for each hit (deduped within the same drag gesture); in `client/src/hooks/useDrawing.js`
- [ ] T047 [US5] Add Ctrl/Cmd+Z keyboard shortcut for undo: register `keydown` listener in `Toolbar.jsx` or `WhiteboardRoom.jsx`; call `onUndo()` when shortcut fires and `undoStack.length > 0`; emit `stroke:undo { operationId, strokeId, userId }` from `useRoom.undo()`; in `client/src/components/Toolbar/Toolbar.jsx`
- [ ] T048 [US5] Integrate `Toolbar` into `WhiteboardRoom`: replace toolbar placeholder with `<Toolbar activeTool={activeTool} undoStack={undoStack} onToolChange={setActiveTool} onUndo={undo} />`; propagate `activeTool` to `<Canvas>`; in `client/src/components/WhiteboardRoom/WhiteboardRoom.jsx`

**Checkpoint**: User Story 5 is fully functional and independently testable. All three tools work. Undo is disabled correctly. Zero one-off button variants exist.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: End-to-end validation, accessibility audit, bundle budget check, and logging coverage verification.

- [ ] T049 [P] Write E2E test: two-tab collaboration — Tab A joins room, Tab B joins same room, Tab A draws a stroke, assert Tab B renders each segment before pointer-up, assert `stroke:committed` finalises on both tabs; in `client/tests/e2e/collaboration.spec.js`
- [ ] T050 [P] Write E2E test: reconnection + offline queue flush — Tab A joins and draws, simulates network drop (`page.setOfflineMode(true)`), draws more strokes, restores network, asserts all strokes appear in Tab B within 3 s; in `client/tests/e2e/reconnection.spec.js`
- [ ] T051 [P] Audit all interactive controls (`ToolButton`, `LandingPage` form inputs, `ConnectionStatus` Retry) for WCAG 2.1 AA: minimum 4.5:1 text contrast, `aria-label` on icon-only buttons, `role="status"` on live regions; fix any violations across affected component files
- [ ] T052 [P] Validate gzipped client bundle ≤250 KB: run `npm run build`, check Vite output; if over budget add `build.rollupOptions.output.manualChunks` to split `socket.io-client` and React into separate chunks; update `client/vite.config.js` as needed
- [ ] T053 [P] Verify FR-032 structured logging coverage: confirm `server/src/eventHandlers.js` and `server/src/roomService.js` emit `logger.info` for `room:created`, `room:destroyed`, `user:joined`, `user:left`; confirm `logger.error` fires for unhandled errors; add any missing log calls
- [ ] T054 Run full `quickstart.md` validation: `npm install` in both packages, `npm run dev` for server + client, run `npm test` for both packages, run `npm run test:e2e`, manually execute all User Story acceptance scenarios from `spec.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion — **BLOCKS all user stories**
- **User Stories (Phases 3–7)**: All depend on Phase 2 completion
  - US1 (Phase 3) and US2 (Phase 4) are both P1 and can proceed in parallel once Phase 2 is done
  - US4 (Phase 5) depends on Phase 2 only; can start once Phase 2 is done
  - US3 (Phase 6) and US5 (Phase 7) depend on Phase 2 only; both are P3 and can proceed in parallel
- **Polish (Phase 8)**: Depends on all desired user stories being complete; E2E tests require server + client running

### User Story Dependencies

| Story | Phase | Priority | Depends on | Can run in parallel with |
|-------|-------|----------|------------|--------------------------|
| US1 — Room Creation & Joining | 3 | P1 | Phase 2 | US2, US4 |
| US2 — Real-Time Drawing | 4 | P1 | Phase 2 | US1, US4 |
| US4 — Connection Status | 5 | P2 | Phase 2 + US1 (useRoom) | US3, US5 |
| US3 — Participants Panel | 6 | P3 | Phase 2 + US1 | US5 |
| US5 — Tool Selection | 7 | P3 | Phase 2 + US2 | US3 |

### Within Each User Story

1. Test tasks MUST be written and confirmed **failing** before implementation tasks start
2. Within implementation: services/hooks before components; hooks before integration
3. Story complete and checkpoint validated before moving to the next priority

---

## Parallel Execution Examples

### Phase 2: Foundational (run T008–T010 together)

```bash
# These three tasks operate on separate files and have no cross-dependencies:
Task T008: Define EVENT_NAMES and ERROR_CODES in server/utils/constants.js
Task T009: Define EVENT_NAMES and perf constants in client/src/utils/constants.js
Task T010: Implement structured JSON logger in server/utils/logger.js
```

### Phase 3: User Story 1 (run T017–T020 together after phase confirmation)

```bash
# Tests (write and confirm failing):
Task T017: Write LandingPage unit tests in client/src/components/LandingPage/LandingPage.test.jsx
Task T018: Write room join integration tests in client/tests/integration/roomFlow.test.js

# Implementations (after tests are confirmed failing):
Task T019: Implement LandingPage component in client/src/components/LandingPage/LandingPage.jsx
Task T020: Implement useRoom hook in client/src/hooks/useRoom.js
```

### Phase 4: User Story 2 (run T023–T027 together)

```bash
# Tests (write and confirm failing):
Task T023: Write drawingEngine unit tests in client/tests/unit/drawingEngine.test.js
Task T024: Write useCanvas unit tests in client/tests/unit/useCanvas.test.js
Task T025: Write Canvas component unit tests in client/src/components/Canvas/Canvas.test.jsx

# Implementations (after tests are confirmed failing):
Task T026: Implement drawingEngine.js in client/src/services/drawingEngine.js
Task T027: Implement canvasService.js in client/src/services/canvasService.js
```

### Phase 5: User Story 4 (run T033–T035 together)

```bash
# Tests (write and confirm failing):
Task T033: Write useConnection unit tests in client/tests/unit/useConnection.test.js
Task T034: Write ConnectionStatus unit tests in client/src/components/ConnectionStatus/ConnectionStatus.test.jsx

# Implementation (after tests confirmed failing):
Task T035: Implement useConnection hook in client/src/hooks/useConnection.js
```

---

## Implementation Strategy

### MVP First (User Stories 1 + 2 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational — **CRITICAL, blocks everything**
3. Complete Phase 3: User Story 1 (Room Creation and Joining)
4. **STOP and VALIDATE**: Two users can join the same room and see canvas state
5. Complete Phase 4: User Story 2 (Real-Time Drawing)
6. **STOP and VALIDATE**: Two users draw in real time, strokes sync point-by-point
7. **Deploy / demo** the P1 MVP

### Incremental Delivery

1. Phase 1 + 2 → Foundation ready
2. + Phase 3 (US1) → Room join/hydration → **Demo #1**
3. + Phase 4 (US2) → Real-time drawing → **Demo #2** (core product)
4. + Phase 5 (US4) → Connection resilience → **Demo #3**
5. + Phase 6 (US3) → Participants panel → **Demo #4**
6. + Phase 7 (US5) → Tool selection → **Demo #5** (feature-complete MVP)
7. Phase 8: Polish → **Production-ready**

### Parallel Team Strategy

With multiple developers (after Phase 2 completes):

- **Developer A**: Phase 3 (US1) — LandingPage, useRoom, WhiteboardRoom shell
- **Developer B**: Phase 4 (US2) — drawingEngine, useCanvas, Canvas component
- **Developer C**: Phase 5 (US4) — useConnection, ConnectionStatus, reconnection flow

Once all P1/P2 stories merge, each developer picks a P3 story independently.

---

## Notes

- `[P]` tasks operate on distinct files and have no dependencies on incomplete tasks — safe to parallelize
- `[Story]` label maps each task to a specific user story for traceability and independent delivery
- Each story's **Independent Test** section defines the verification criteria before moving to the next priority
- All test tasks must be written and confirmed failing **before** the corresponding implementation tasks begin (Constitution Principle I)
- Commit after each completed task or logical group; use `speckit.git.commit` for auto-commit hooks
- Stop at each **Checkpoint** to validate the story independently before proceeding
