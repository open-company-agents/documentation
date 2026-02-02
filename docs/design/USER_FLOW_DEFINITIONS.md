# User Flow Definitions

**Pivot Note (2026-02-02):** Updated for Cloud-Native SaaS Model. No local installation required.

## High-Level User Journey

The user journey follows a standard SaaS lifecycle: **Acquisition -> Onboarding -> Consumption -> Retention**.

### 1. Onboarding (The "Zero Install" Experience)
*   **Goal:** Get the user from "Landing Page" to "First Agent Response" in < 2 minutes.
*   **Steps:**
    1.  **Landing Page:** Value prop. "Start Free Trial".
    2.  **Sign Up / Login:** Email/Password or OAuth (Google/GitHub).
    3.  **Onboarding Wizard:**
        *   "What do you want to build?" (Select use case template).
        *   "Connect Accounts" (Optional: Connect GitHub/Linear integration).
    4.  **Dashboard:** User lands in the main control center.

### 2. Provisioning & Billing
*   **Context:** Unlike local tools, compute costs money. Users need credits.
*   **Flow:**
    1.  **Free Tier:** New users get N free credits (enough for ~5 demo sessions).
    2.  **Top-up:** User clicks "Add Credits" -> Stripe Checkout -> Balance Updated.
    3.  **Usage Monitor:** Dashboard shows a "Fuel Gauge" of remaining credits.

### 3. Core Loop: Running an Agent
*   **Interface:** Web-based Chat / Terminal hybrid.
*   **Flow:**
    1.  **Initiate:** User clicks "New Agent Session".
    2.  **Configuration:** Select Model (GPT-4/Claude), select Environment (Dev/Research).
    3.  **Booting:** UI shows "Provisioning Cloud Agent..." (Spinner/Progress bar).
    4.  **Active Session:**
        *   User types natural language goal.
        *   Remote Agent (Fargate) processes goal.
        *   UI streams logs, thoughts, and terminal output in real-time.
        *   *Feature:* **Cloud Browser View.** If the agent browses the web, the UI mirrors the headless browser view.
    5.  **Completion/Termination:**
        *   Agent finishes task.
        *   Container spins down.
        *   Session logs saved to history.

### 4. Review & Artifacts
*   **Context:** What did the agent produce?
*   **Flow:**
    1.  User visits "History" tab.
    2.  Selects a past session.
    3.  Views generated files (code, PDFs, reports) stored in S3.
    4.  Downloads artifacts or exports to external service (e.g., "Push to GitHub").

## Critical UX Changes (vs. Old Model)

*   **Latency Expectation:** Users must understand there is a "Boot time" for the cloud container. We need nice loading states (skeleton screens, "Waking up your agent...").
*   **Billing Visibility:** Money is constantly being drained while running. The UI must clearly show "Session Cost: $0.12" or similar to prevent bill shock.
*   **Persistence:** Clarify that the *environment* is ephemeral. "Don't save files to the desktop, they will vanish. Use the Project Folder."

## Persona: "The Cloud Manager"
*   **User:** Product Manager or Dev Lead.
*   **Goal:** Automate Jira ticket analysis without configuring Python environments locally.
*   **Pain Point:** "I don't want to install Docker."
*   **Solution:** OpenClaw SaaS handles the Docker part. They just login and chat.
