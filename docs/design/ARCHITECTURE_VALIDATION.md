# Architecture Validation: Centralized SaaS Platform

## Executive Summary

**Pivot Alert (2026-02-02):** The architecture has shifted from a local/hybrid tool to a pure **Centralized SaaS Platform**.
We are building a cloud-native platform where users sign up, purchase credits, and deploy agents in a managed environment (AWS). Nothing runs on the user's local machine.

## Core Architecture: Cloud-Native SaaS

The system is designed as a fully managed service running on AWS.

### 1. High-Level Components

*   **Frontend (Client):** Single Page Application (React/Next.js) hosted on AWS S3 + CloudFront. Serves as the dashboard for user management, billing, and agent interaction.
*   **Core API (Backend):** Centralized Node.js API (AWS Lambda or Fargate) handling authentication, billing, agent orchestration, and state management.
*   **Agent Runtime (Compute):** Ephemeral, isolated containers running on AWS ECS (Fargate). Each agent session spins up a dedicated container.
*   **Data Layer:** Centralized persistence for user data, session logs, and billing ledger (AWS RDS/PostgreSQL or DynamoDB).

### 2. Detailed Stack

| Component | Technology | Role |
| :--- | :--- | :--- |
| **Frontend** | React, S3, CloudFront | User Interface, Dashboard, Web Terminal. |
| **Auth** | Cognito / Custom JWT | User Identity & Access Management. |
| **API Gateway** | AWS API Gateway | Entry point for REST/WebSocket traffic. |
| **Core API** | Node.js (Lambda/Fargate) | Business logic: User mgmt, Credit deduction, Orchestration. |
| **Orchestrator** | AWS Step Functions / SQS | Managing agent lifecycle (provisioning, termination). |
| **Agent Runtime** | AWS Fargate (ECS) | **The Engine.** Runs the OpenClaw agent process in isolation. |
| **Database** | Postgres (RDS) or DynamoDB | Users, Teams, Billing Ledger, Session History. |
| **Logs/Stream** | CloudWatch / WebSocket | Real-time streaming of agent output to the frontend. |
| **Billing** | Stripe Integration | Payment processing and credit purchase. |

## Key Technical Decisions

### A. The "No-Localhost" Policy
*   **Constraint:** Users do not install CLI tools or run agents locally.
*   **Solution:** All execution happens in the cloud. The web interface is the only required access point.

### B. Agent Isolation (Runtime)
*   **Constraint:** Multi-tenant security is critical. User A's agent cannot touch User B's data.
*   **Solution:** **One Container Per Session.** We use AWS Fargate to spin up a fresh, sandboxed environment for every agent run.
    *   *Pros:* Hard isolation, scalable, serverless (no EC2 management).
    *   *Cons:* Cold start times (requires optimization or warm pools).

### C. Billing & Credits
*   **Constraint:** B2B/B2C SaaS model.
*   **Solution:** Pre-paid Credit System.
    *   User buys 1000 credits via Stripe.
    *   Running an agent costs X credits/minute + Y credits/token.
    *   Core API enforces limits and deducts credits in near real-time.

### D. Data Persistence
*   **Constraint:** Centralized history.
*   **Solution:** Agents write logs and artifacts to S3/Database immediately. State is not local to the container; it is externalized to ensure durability after the container dies.

## Validated Flows

### 1. The "Launch" Flow
1.  User logs into Web Dashboard.
2.  User clicks "New Session".
3.  **Frontend** calls **Core API** (`POST /sessions`).
4.  **Core API** checks credit balance.
5.  **Core API** triggers **ECS Fargate** task definition.
6.  **Agent Container** boots up (~30s-60s).
7.  **Agent** connects back via WebSocket to the central hub.
8.  User sees "Connected" in the browser and starts chatting.

### 2. The "Billing" Flow
1.  Agent consumes resources (LLM tokens, runtime seconds).
2.  **Agent Runtime** reports usage to **Core API** periodically (heartbeat).
3.  **Core API** deducts credits from User's ledger in DB.
4.  If credits <= 0, Core API sends `SIGTERM` to the container.

## Security Considerations

*   **Network Isolation:** Agent containers run in a private subnet with restricted outbound access (allow-listed domains only).
*   **Ephemeral Filesystem:** Any data written to disk in Fargate is lost upon termination unless explicitly saved to the user's S3 bucket.
*   **Secrets Management:** API keys for external tools are injected via AWS Secrets Manager at runtime, scoped strictly to that session.

## Next Steps

1.  Prototype the Fargate task definition for the Agent Runtime.
2.  Design the DynamoDB schema for the Credit Ledger.
3.  Implement the WebSocket relay for real-time terminal streaming to the browser.
