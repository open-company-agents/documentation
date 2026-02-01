# Financial Viability of SaaS Stack

## Executive Summary
For a $5/month user price point, the proposed stack faces significant margin challenges, primarily driven by payment processing fixed fees.

## 1. Auth: Clerk
*   **Status:** **VIABLE**
*   **Free Tier:** Generous 10,000 Monthly Active Users (MAUs).
*   **Growth Cost:** After 10k, cost is ~$0.02 per MAU on Pro Plan ($25/mo base).
*   **Impact:** Negligible (<1% of revenue) until massive scale.

## 2. Payments: Lemon Squeezy (The Bottleneck)
*   **Status:** **HIGH RISK** for $5 price point.
*   **Fee Structure:** 5% + $0.50 per transaction.
*   **Scenario A ($5.00 Charge):**
    *   Fee: $0.25 (5%) + $0.50 = $0.75
    *   Effective Fee Rate: **15%**
    *   Net Revenue: $4.25
*   **Recommendation:**
    *   **Option 1:** Raise minimum top-up/subscription to **$10.00**.
        *   Fee: $0.50 + $0.50 = $1.00 (10% effective).
    *   **Option 2:** Use annual billing ($60/year) to reduce transaction count.

## 3. Email: Resend
*   **Status:** **CONDITIONAL**
*   **Free Tier:** 100 emails/day (3,000/mo). Strict daily limit.
*   **Pro Plan:** Starts at ~$20/mo for higher volume.
*   **Impact:** For a Beta with <100 daily active users sending invites/alerts, Free is fine. For marketing blasts, we will hit the ceiling immediately.
*   **Recommendation:** Use "System Emails" only (auth/receipts) on Resend. Move marketing to a cheaper bulk provider if needed later.

## Conclusion
We cannot profitably sustain $5 micropayments with Lemon Squeezy's $0.50 fixed fee. **Action Required:** Change pricing strategy to minimum $10 purchase or switch payment provider for low-ticket items.
