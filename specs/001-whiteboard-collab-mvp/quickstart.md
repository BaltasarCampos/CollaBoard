# Quickstart: Whiteboard Collaboration MVP

**Branch**: `001-whiteboard-collab-mvp` | **Date**: 2026-05-10

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Node.js | ≥22 LTS | `node --version` to verify |
| npm | ≥10 | Bundled with Node.js 22 |
| Git | any | For branch management |

No database, no Docker, no external services required for local development.

---

## Repository Layout

```text
CollaBoard/
├── client/          # React SPA (Vite)
├── server/          # Node.js + Express + Socket.IO
└── specs/           # Specification and planning documents
```

---

## Initial Setup

Clone the repository (if you haven't already) and install dependencies for both packages:

```bash
git clone <repo-url> CollaBoard
cd CollaBoard

# Install server dependencies
cd server && npm install && cd ..

# Install client dependencies
cd client && npm install && cd ..
```

---

## Running in Development

Open **two terminal windows** from the repository root.

**Terminal 1 — Server** (port 3001 by default):
```bash
cd server
npm run dev
```

Expected output:
```
[server] Listening on http://localhost:3001
[server] Socket.IO ready
```

**Terminal 2 — Client** (port 5173 by default, via Vite):
```bash
cd client
npm run dev
```

Expected output:
```
  VITE v5.x.x  ready in xxx ms
  ➜  Local:   http://localhost:5173/
```

Open [http://localhost:5173](http://localhost:5173) in your browser. To test collaboration, open a second tab or a second browser window with the same URL and join the same room code.

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3001` | Socket.IO / Express server port |
| `CLIENT_ORIGIN` | `http://localhost:5173` | CORS origin allowed by the server |
| `HYDRATE_CHUNK_THRESHOLD_BYTES` | `51200` | Canvas state size (bytes) above which chunked hydration is used |

Create a `server/.env` file to override defaults:
```
PORT=3001
CLIENT_ORIGIN=http://localhost:5173
```

Create a `client/.env` file (Vite reads `VITE_` prefixed variables):
```
VITE_SERVER_URL=http://localhost:3001
```

---

## Running Tests

### Unit + Integration Tests (Jest)

```bash
# Server tests
cd server && npm test

# Client tests
cd client && npm test
```

To run in watch mode during development:
```bash
npm run test:watch
```

### End-to-End Tests (Playwright)

E2E tests require both the server and client dev servers to be running (see above).

```bash
cd client
npm run test:e2e
```

The Playwright configuration (`client/playwright.config.js`) targets `http://localhost:5173` and launches multi-tab scenarios to validate real-time collaboration.

To run with a visible browser (headed mode) for debugging:
```bash
npm run test:e2e -- --headed
```

### Running All Tests

```bash
# From repository root (requires both packages to have test scripts)
npm run test --workspaces
```

---

## npm Scripts Reference

### `server/package.json`

| Script | Command | Description |
|--------|---------|-------------|
| `dev` | `node --watch src/server.js` | Dev server with auto-restart on file changes |
| `start` | `node src/server.js` | Production server (no watch) |
| `test` | `jest` | Run all server tests |
| `test:watch` | `jest --watch` | Watch mode |

### `client/package.json`

| Script | Command | Description |
|--------|---------|-------------|
| `dev` | `vite` | Vite dev server with HMR |
| `build` | `vite build` | Production build (output to `client/dist/`) |
| `preview` | `vite preview` | Serve production build locally |
| `test` | `jest` | Run all client unit/integration tests |
| `test:watch` | `jest --watch` | Watch mode |
| `test:e2e` | `playwright test` | End-to-end tests |

---

## Development Workflow (TDD)

Per [Constitution Principle I](../../../.specify/memory/constitution.md), all implementation follows a **test-first** cycle:

1. **Write a failing test** for the smallest next behaviour.
2. **Get approval** (review the test intent with a teammate or self-review).
3. **Make it pass** with the minimal implementation.
4. **Refactor** without breaking the test.
5. **Commit** with a message referencing the relevant task ID (e.g., `feat(canvas): local stroke rendering [task-003]`).

### Feature Branch

All work for this feature lives on branch `001-whiteboard-collab-mvp`.

```bash
git checkout 001-whiteboard-collab-mvp
```

---

## Bundle Budget Check

The client build MUST produce a gzipped initial bundle ≤250 KB (FR-026). After building:

```bash
cd client && npm run build
```

Vite outputs bundle sizes. To check gzipped sizes explicitly:

```bash
# Install if not already present
npm install -g gzip-size-cli

gzip-size client/dist/assets/*.js
```

If the budget is exceeded, investigate with:
```bash
npx vite-bundle-visualizer
```

---

## Useful Local Testing Scenarios

### Two-user collaboration
1. Open [http://localhost:5173](http://localhost:5173) in Tab A → enter name "Alice", leave room code blank → submit.
2. Copy the room code from the URL or UI.
3. Open [http://localhost:5173](http://localhost:5173) in Tab B → enter name "Bob", paste the room code → submit.
4. Draw in Tab A — verify Tab B renders strokes incrementally.

### Reconnection resilience (SC-007, SC-009)
1. Join a room in two tabs and draw some strokes.
2. In Tab A, open DevTools → Network → set throttling to "Offline".
3. Draw more strokes in Tab A (they queue locally).
4. Re-enable the network.
5. Verify the queued strokes appear in Tab B within 3 seconds.

### Mobile viewport
1. Open DevTools → Device toolbar → set to "iPhone SE" (375 × 667 px).
2. Verify the landing page form is fully usable and the whiteboard is functional.
