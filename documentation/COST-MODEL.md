# Cost Model & Pricing Analysis - Q1 2025
**Status:** DRAFT
**Author:** Business Analyst (OpenClaw)
**Date:** 2026-02-01

## 1. Infrastructure Costs (AWS us-east-1)

### Compute: AWS Fargate
Precios estimados para `us-east-1`. Se asume un agente "medio" con 0.5 vCPU y 1GB RAM.

| Recurso | Tipo | Precio Unitario (aprox) | Coste Mensual (24/7) | Ahorro Spot |
| :--- | :--- | :--- | :--- | :--- |
| **vCPU** | On-Demand | $0.04048 / vCPU-h | $29.15 | - |
| **vCPU** | **Spot** | ~$0.01214 / vCPU-h (Avg) | **$8.74** | ~70% |
| **Memoria** | On-Demand | $0.004445 / GB-h | $3.20 | - |
| **Memoria** | **Spot** | ~$0.00133 / GB-h (Avg) | **$0.96** | ~70% |
| **TOTAL** | **On-Demand** | **-** | **$32.35** | **-** |
| **TOTAL** | **Spot** | **-** | **$9.70** | **~70%** |

**Recomendación:** Fargate Spot es mandatorio para rentabilidad. El riesgo de interrupción es bajo para cargas de trabajo stateless o resilientes.

### Costes Ocultos / Compartidos

*   **NAT Gateway:**
    *   Coste Fijo: $0.045/h (~$32.40/mes).
    *   Procesamiento: $0.045/GB.
    *   *Estrategia:* **COMPARTIDO**. Un solo NAT Gateway puede servir a cientos de agentes en la misma VPC. Coste por agente despreciable si N > 50.
*   **Data Transfer Out:** ~$0.09/GB (primeros 10TB). Asumiendo tráfico de texto/JSON, es bajo (<$1/mes/agente).
*   **CloudWatch Logs:**
    *   Ingestion: $0.50/GB.
    *   Storage: $0.03/GB.
    *   *Control:* Configurar retención agresiva (e.g., 7-14 días) para evitar costes acumulativos.

## 2. LLM Costs (OpenRouter / APIs)

Estimación basada en precios de mercado Q4 2024 / Q1 2025.

### Modelos "Smart" (Reasoning/Coding)
*Ej: Claude 3.5 Sonnet, GPT-4o*
*   **Input:** ~$3.00 / 1M tokens
*   **Output:** ~$15.00 / 1M tokens

### Modelos "Fast" (Chat/Routine)
*Ej: Claude 3.5 Haiku, Gemini 1.5 Flash*
*   **Input:** ~$0.25 / 1M tokens
*   **Output:** ~$1.25 / 1M tokens

## 3. Frontend & Edge

*   **Vercel Pro:** $20/usuario/mes.
    *   Ideal para equipos pequeños/medianos y velocidad de desarrollo.
*   **AWS CloudFront + S3 (Self-hosted):**
    *   Coste base cercano a $0. Paga por GB transferido (~$0.085/GB).
    *   Mucho más barato a escala, pero requiere mantenimiento DevOps.

---

## 4. Pay-As-You-Go Model (Strategic Pivot)

**Context:** The fixed $49/mo subscription was identified as a high barrier to entry. We are pivoting to a prepaid credit system ("Pay-as-you-go").

### Unit Pricing Structure

1.  **Compute Runtime:**
    *   **Price:** **$0.035 / active hour**.
    *   *Cost Basis:* ~$0.013-0.015/h (Fargate Spot + Networking/NAT share).
    *   *Margin:* ~57% on compute. Covers idle overheads, orchestration, and "scale to zero" inefficiencies.
    *   *Behavior:* Billing stops when the agent is paused/stopped.

2.  **Intelligence (Tokens):**
    *   **Price:** **Provider Cost + 20%**.
    *   *Mechanism:* Real-time metering based on OpenRouter/API usage.
    *   *Rationale:* 20% margin covers token processing overhead, logging, and payment processing fees (Stripe).

### Break-Even Analysis

With the removal of fixed subscriptions, we must ensure "active" users cover their baseline footprint.

*   **Fixed Baseline Costs (per active user/month):**
    *   Log Storage (CloudWatch/S3): ~$0.50 (Hot storage).
    *   Database State (DynamoDB): ~$0.20.
    *   Shared Infrastructure (NAT/ALB): ~$0.30.
    *   **Total Baseline:** **~$1.00 / month** if the user exists and has data.
*   **Break-Even Point:**
    *   A user needs to consume **~$2.50 in credits per month** to be profitable (assuming ~40% blended margin).
    *   This equals approx **10-15 hours of active agent work** OR **~500k "Fast" tokens**.

### Free Tier Feasibility ($1.00 Credit)

Giving $1.00 in free credits is a low-risk User Acquisition strategy.

**What does $1.00 buy?**
*   **Scenario A (Exploration):** ~28 hours of runtime (idle/thinking) using minimal tokens.
*   **Scenario B (Heavy Coding - Smart Models):**
    *   2 Hours Runtime ($0.07).
    *   40k Tokens (Claude 3.5 Sonnet @ ~$15/M In+Out blended): ~$0.60.
    *   **Result:** ~2.5 hours of intense "Senior Dev" pair programming.
*   **Conclusion:** $1.00 is sufficient for a "Wow" moment, allowing a user to complete 1 meaningful task (e.g., "Refactor this file" or "Write a test suite").

### Recommendation
Implement a prepaid wallet system.
*   **Min Top-up:** $5.00.
*   **Auto-pause:** Agents suspend automatically when balance < $0.
*   **Expiration:** Credits expire after 12 months (GAAP revenue recognition).
*   **Enterprise:** Volume discounts available for usage > $500/mo.
