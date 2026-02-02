# Project Roadmap & Backlog

**Last Updated:** 2026-02-02
**Status:** ACTIVE
**Owner:** Orion (PO/PM)

This document tracks the development tasks required to build the OpenClaw Agent Dashboard, based on the approved Architecture and UX definitions.

## 🎯 Phase 1: MVP (Foundation & Core Loop)
*Goal: A working local dashboard where users can spawn an agent and see it think in real-time.*

### 1. Infrastructure & Scaffolding
- [ ] **Frontend Init:** Initialize React + Vite project with TypeScript.
    - [ ] Configure TailwindCSS.
    - [ ] Setup `shadcn/ui` component library.
    - [ ] Setup `Zustand` for state management.
    - [ ] Setup `TanStack Query` for API data fetching.
- [ ] **Backend Init:** Initialize Node.js + Fastify project (TypeScript).
    - [ ] Setup basic HTTP server structure.
    - [ ] Configure `Socket.io` (or `fastify-websocket`) for real-time events.
- [ ] **Database Setup:**
    - [ ] Initialize Prisma ORM.
    - [ ] Define Schema v1: `Agent`, `Run`, `Workspace` (default hardcoded), `Secret`.
    - [ ] Configure SQLite for local persistence.

### 2. Backend Core (The Orchestrator)
- [ ] **Agent Supervisor Service:**
    - [ ] Implement process spawning mechanism (child_process/worker_threads).
    - [ ] Create abstraction for "Agent Lifecycle" (Start, Stop, Pause).
- [ ] **Real-time Logging System:**
    - [ ] Implement log buffering (ring-buffer) for late-joiners.
    - [ ] Emit events: `agent:thought`, `agent:action`, `agent:error` via WebSocket.
- [ ] **API Endpoints:**
    - [ ] `POST /agents`: Create new agent config.
    - [ ] `GET /agents`: List available agents.
    - [ ] `POST /agents/:id/run`: Trigger execution.
    - [ ] `GET /history`: Retrieve past runs from DB.

### 3. Frontend Core (The Dashboard)
- [ ] **Agent Creation Wizard:**
    - [ ] Form to define Name, Model, and Tools.
    - [ ] Basic validation.
- [ ] **Live Console View:**
    - [ ] Terminal-like component to display streaming logs.
    - [ ] Connection status indicator (WebSocket health).
- [ ] **Agent List:** Simple card view of configured agents.

### 4. Security (Basic)
- [ ] **Secrets Management:**
    - [ ] DB table for encrypted API keys (AES-256).
    - [ ] Backend logic to inject decrypted keys into Agent env at runtime.
    - [ ] UI to input keys (never display back in plain text).

---

## 🚀 Phase 2: Refinement & Scale
*Goal: Robustness, team features, and advanced observability.*

- [ ] **Multi-tenancy:** Enable `workspace_id` logic in UI/API (currently schema-only).
- [ ] **Auth Strategy:** Implement OAuth2 / OIDC for hosted environments.
- [ ] **Advanced Visualization:** Graphs for token usage and cost tracking.
- [ ] **Drag-and-Drop Editor:** Visual flow builder for agent logic (per UX recommendation).
- [ ] **Template Library:** Pre-built agent configurations (e.g., "Log Analyzer").
