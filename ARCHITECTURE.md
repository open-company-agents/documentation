# Propuesta Técnica: OpenClaw Cloud (MVP)

> **Estado:** Draft
> **Autor:** Stratus (Cloud Architect)
> **Fecha:** 2026-01-31

## 1. High-Level Design

El siguiente diagrama ilustra el flujo de interacción desde que un usuario solicita una instancia hasta que el agente está operativo. El diseño prioriza la separación de control (Control Plane) y ejecución (Data Plane).

```mermaid
sequenceDiagram
    participant U as Usuario Web
    participant API as API Gateway + Lambda
    participant DB as DynamoDB (State)
    participant O as Orchestrator (Fargate)
    participant A as Agente (Firecracker/ECS)

    U->>API: POST /agents/spawn (Token)
    API->>DB: Crea registro "Pending"
    API->>O: Evento: "Provision Agent"
    O->>A: Boot Instance (con Sandbox)
    activate A
    A->>A: Carga Runtime + Herramientas
    A-->>O: Health Check OK (IP/Port)
    O->>DB: Actualiza estado "Running"
    O-->>U: Retorna WebSocket URL
    deactivate A

    U->>A: Conexión WS directa (Auth Token)
    Note right of U: Interacción en tiempo real\n(Comandos, Logs, Terminal)
```

## 2. Tecnología de Aislamiento: Recomendación

Para el MVP, la prioridad es **balancear seguridad, coste y facilidad de implementación**.

### Opción A: AWS ECS (Fargate)
*   **Pros:** Gestión cero de servidores, integración nativa, seguridad por diseño (cada tarea tiene su kernel).
*   **Contras:** Tiempo de arranque "lento" (30-60s), coste mínimo fijo por tarea, difícil de pausar/reanudar instantáneamente.

### Opción B: AWS Lambda (Serverless)
*   **Pros:** Arranque instantáneo, escala a cero.
*   **Contras:** Tiempo máximo de ejecución (15 min), sin persistencia de estado local fácil, Websockets requieren gestión externa (API Gateway V2). **No viable para sesiones largas de agentes.**

### Opción C: Firecracker (MicroVMs) sobre EC2 Bare Metal
*   **Pros:** Aislamiento duro (KVM), arranque en milisegundos (<150ms), máxima densidad.
*   **Contras:** Complejidad operativa alta (gestionar fleet de EC2, networking manual).

### 🏆 Recomendación MVP: **AWS ECS con Fargate Spot**
Aunque Firecracker es el "sueño arquitectónico" para el futuro (fase Scale-up), para el MVP necesitamos validar mercado sin gestionar infraestructura base.

*   **Por qué:** Nos permite lanzar contenedores aislados sin configurar servidores.
*   **Optimización de Costos:** Usaremos **Fargate Spot** (hasta 70% de descuento) para las cargas de trabajo de los agentes, ya que podemos tolerar interrupciones (o manejarlas con reintentos).
*   **Seguridad:** Cada agente corre en su propia ENI (Interfaz de Red) y Sandbox de Fargate.

## 4. Cost Optimization & Scale-to-Zero Strategy

Bajo el modelo "Pay-as-you-go", el objetivo es que el coste de infraestructura sea linealmente proporcional al uso real. **Si nadie usa el agente, el coste debe ser cercano a $0.**

### Estrategia: "Scale-to-Zero" Estricto
Los agentes no son demonios de larga duración (24/7). Son efímeros.
1.  **Idle Timeout:** Si un agente no recibe comandos en X minutos (e.g., 15 min), el contenedor se termina.
2.  **State Hydration:** El estado (memoria, ficheros) se persiste en S3/EFS antes de morir. Al revivir, se "hidrata" el nuevo contenedor.

### El Dilema del "Cold Start" (Validación Técnica)
El principal trade-off de Fargate en un modelo Scale-to-Zero es el tiempo de arranque.

| Tecnología | Cold Start | Coste Idle | Complejidad Ops |
| :--- | :--- | :--- | :--- |
| **AWS Fargate** | ~30-60s | $0 (si apagado) | Baja (Serverless) |
| **Firecracker (Fly.io)** | < 500ms | $0 (si apagado) | Media (Vendor specific) |
| **Warm Pool (Fargate)** | < 2s | $$ (siempre encendido) | Media (Gestión de pool) |

### 🧠 Veredicto de Arquitectura: Fargate Spot (MVP)

A pesar de que 45s de espera es una fricción UX significativa, **mantenemos Fargate Spot para el MVP** por las siguientes razones:

1.  **Simplicidad Operativa:** No queremos gestionar clusters de Kubernetes ni VMs bare metal en la fase 1.
2.  **Mitigación UX:** Se implementará una pantalla de carga ("Despertando a tu Agente... ☕") que es aceptable para herramientas profesionales de uso por sesión.
3.  **Roadmap:**
    *   **Fase 1 (MVP):** Fargate Spot. Cold start ~45s.
    *   **Fase 2 (Optimización):** Mantener un "Hot Pool" de 2-3 agentes pre-calentados para usuarios Premium.
    *   **Fase 3 (Ideal):** Migrar a orquestación basada en Firecracker (vía Fly.io machines API o AWS Bare Metal) para arranques sub-segundo.

## 5. Estimación de Costos (100 Agentes 24/7)

Supuesto: 100 agentes corriendo continuamente durante un mes (730 horas).
Recursos por Agente: 0.5 vCPU, 1 GB RAM (Suficiente para MVP de Node.js/Python scripts).

### Cálculo con Fargate Spot (us-east-1)
*   **vCPU:** $0.0125 / vCPU-hora (aprox Spot)
*   **Memoria:** $0.0015 / GB-hora (aprox Spot)

**Costo por Agente/Hora:**
(0.5 * $0.0125) + (1 * $0.0015) = **$0.00775 / hora**

**Costo Mensual (1 Agente):**
$0.00775 * 730 horas = **$5.66**

**Costo Mensual (100 Agentes):**
$5.66 * 100 = **$566 USD / mes**

> *Nota: Esto es solo cómputo. No incluye transferencia de datos, DynamoDB ni API Gateway (que suelen ser marginales comparado con cómputo 24/7).*

### Comparativa vs EC2 On-Demand (t4g.small)
*   ~ $12.00 / mes por instancia.
*   Ahorro con Fargate Spot: ~50% y sin gestión de OS.

---
**Conclusión:** Podemos mantener una flota de 100 agentes activos por menos de $600/mes usando Fargate Spot. El modelo de negocio debe cubrir holgadamente estos $6 por agente/mes.
