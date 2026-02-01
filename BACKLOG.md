# Product Backlog - MVP Sprint 1

**Project:** AI Cloud Environment (MVP)
**Sprint Goal:** Launch functional Dashboard with Terminal Loading, Chat, and usage-based Billing ($0.05/hr).

---

## 📂 Epic 1: Core Infra & API (Backend)
**Description:** Foundation for provisioning and managing ephemeral AI environments on AWS.

### 📝 User Story: Provisioning API (Fargate Spot)
**As a** System,
**I want to** provision ECS Fargate Spot tasks via an API endpoint,
**So that** users get a dedicated environment upon request.

**Acceptance Criteria:**
- [ ] Endpoint `POST /api/v1/environments` accepts user ID.
- [ ] Triggers AWS ECS RunTask using Fargate Spot capacity provider.
- [ ] Returns `environment_id` and `initial_status` (PROVISIONING).
- [ ] Injects required environment variables (e.g., AGENT_ID).
- [ ] Failure handling: Retries on Spot unavailability or returns user-friendly error.

### 📝 User Story: Scale-to-Zero Logic
**As a** Cost Manager,
**I want to** automatically terminate environments after a period of inactivity,
**So that** we do not incur unnecessary infrastructure costs.

**Acceptance Criteria:**
- [ ] System monitors active connections/usage (WebSocket/HTTP).
- [ ] "Idle Timer" starts when no activity is detected for 5 minutes.
- [ ] At T+5m idle: Trigger termination sequence.
- [ ] Update status to TERMINATED in DB.
- [ ] Notify user (if connected) or log the event.

---

## 📂 Epic 2: Billing & Credits (Backend)
**Description:** Monetization engine to handle credits and hourly metering.

### 📝 User Story: Wallet System (Ledger)
**As a** User,
**I want to** have a credit balance (Wallet) managed by the system,
**So that** I can pay for the compute time I consume.

**Acceptance Criteria:**
- [ ] Database schema for `Wallets` (User_ID, Balance, Currency).
- [ ] Database schema for `Transactions` (Type: CREDIT/DEBIT, Amount, Timestamp, Reference).
- [ ] API to query current balance.
- [ ] API/Internal function to deduct credits atomically.
- [ ] Prevent provisioning if Balance <= Minimum_Threshold.

### 📝 User Story: Hourly Metering ($0.05/h)
**As a** Finance Controller,
**I want to** charge users $0.05 per hour of active environment usage,
**So that** the business model is sustainable.

**Acceptance Criteria:**
- [ ] Cronjob runs every N minutes (e.g., 5-15 min) to check active environments.
- [ ] Calculate cost: `$0.05 * (duration_in_hours)`.
- [ ] Deduct proportional amount from User Wallet.
- [ ] **Constraint:** Price is fixed at **$0.05/hour** (ignoring lower suggestions).
- [ ] If balance reaches 0: Trigger Scale-to-Zero immediately.

---

## 📂 Epic 3: User Dashboard (Frontend)
**Description:** The user-facing interface for interaction and status.

### 📝 User Story: "Hacker Terminal" Loading Screen
**As a** User waiting for provisioning (45s),
**I want to** see a "Hacker Terminal" style loading screen with scrolling logs,
**So that** the wait time feels like a feature ("immersion") rather than a delay.

**Acceptance Criteria:**
- [ ] Visual style: Green/Amber text on black background, monospaced font.
- [ ] Animation: Fake or real "boot sequence" logs scrolling (e.g., "Initializing kernel...", "Allocating memory...", "Connecting to matrix...").
- [ ] Progress indicator matched to estimated 45s wait time.
- [ ] Transition: Smooth fade-out to Chat Interface once status is READY.

### 📝 User Story: Chat Interface
**As a** User,
**I want to** chat with the AI Agent in a dedicated interface,
**So that** I can instruct it to perform tasks.

**Acceptance Criteria:**
- [ ] Chat window with input area and message history.
- [ ] WebSocket/HTTP polling connection to backend.
- [ ] Display Agent messages clearly (distinct from User messages).
- [ ] Auto-scroll to bottom on new message.
- [ ] Basic Markdown rendering for Agent code blocks.
