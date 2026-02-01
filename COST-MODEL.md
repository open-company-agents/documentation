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

### Consumo Mensual Estimado (Perfil "Medio")
*Supuestos:* 100 interacciones/día, 30 días. 1k tokens in / 500 tokens out por interacción.
*   Total Input: 3M tokens/mes.
*   Total Output: 1.5M tokens/mes.

| Mix de Modelos | Coste Input | Coste Output | **Coste Total LLM** |
| :--- | :--- | :--- | :--- |
| **100% Smart** | $9.00 | $22.50 | **$31.50** |
| **100% Fast** | $0.75 | $1.88 | **$2.63** |
| **Híbrido (20% Smart / 80% Fast)** | $2.40 | $6.00 | **$8.40** |

## 3. Frontend & Edge

*   **Vercel Pro:** $20/usuario/mes.
    *   Ideal para equipos pequeños/medianos y velocidad de desarrollo.
*   **AWS CloudFront + S3 (Self-hosted):**
    *   Coste base cercano a $0. Paga por GB transferido (~$0.085/GB).
    *   Mucho más barato a escala, pero requiere mantenimiento DevOps.

**Decisión:** Iniciar con Vercel (Free/Pro) para velocidad. Migrar a CloudFront si la factura supera los $500/mes.

---

## 4. Unit Economics (Por Agente Activo)

Escenario: Agente 24/7 en Fargate Spot, Mix Híbrido de LLM, Infraestructura compartida.

| Concepto | Coste Mensual Estimado | Notas |
| :--- | :--- | :--- |
| Compute (Spot 0.5vCPU/1GB) | $9.70 | Fargate Spot |
| LLM (Híbrido) | $8.40 | 100 interacciones/día |
| Storage/Logs/DB | $2.00 | DynamoDB, CloudWatch, S3 |
| NAT/Network (Amortizado) | $0.50 | Asumiendo 50+ agentes |
| **COSTO TOTAL (COGS)** | **$20.60** | **Por agente/mes** |

## 5. Pricing Recommendation

Objetivo: **Margen Bruto > 40%**.
Costo Base: ~$20.60

### Tier: "Professional Agent"
*   **Precio Sugerido:** **$49 / mes**
*   **Margen:** $28.40 (58%)
*   **Incluye:**
    *   Agente 24/7.
    *   Hasta 3000 interacciones/mes (Soft limit).
    *   Acceso a modelos "Smart" (limitado/rate-limited).

### Tier: "Enterprise / Power"
*   **Precio Sugerido:** **$99 / mes**
*   **Margen:** $78.40 (79% - asumiendo mismo uso base)
*   **Incluye:**
    *   Mayor memoria/CPU (si necesario).
    *   Prioridad en modelos "Smart".
    *   SLA de soporte.
    *   Retención de logs extendida.

### Observaciones Finales
1.  **Spot es crítico:** Sin Spot, el coste de cómputo salta a ~$32, comiendo todo el margen del plan de $49.
2.  **Control de Tokens:** Es vital implementar *rate limiting* o *budgets* de tokens por usuario para evitar que un solo usuario "heavy" destruya el margen (e.g. loops infinitos con GPT-4o).
3.  **Idle Scaling:** Si el agente no necesita estar activo 24/7 (scale to zero), el coste de cómputo baja drásticamente (a casi $0). Considerar arquitectura *Serverless Lambda* para agentes que no requieren procesos de larga duración.
