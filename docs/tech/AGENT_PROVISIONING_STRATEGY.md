# Estrategia de Aprovisionamiento de Agentes (Agent Provisioning Strategy)

**Estado:** Draft
**Fecha:** 2026-02-02
**Contexto:** Definición técnica de la infraestructura para el despliegue dinámico de agentes OpenClaw en AWS ECS Fargate.

Este documento detalla el ciclo de vida técnico de un Agente OpenClaw, desde la solicitud de creación hasta su ejecución como contenedor aislado en la nube. Nuestro objetivo es garantizar **aislamiento estricto**, **persistencia de datos** y **seguridad en tiempo de ejecución**.

---

## 1. La "Fábrica de Agentes" (High-Level Flow)

El proceso de aprovisionamiento sigue un modelo de "Infraestructura Inmutable" para el código base, pero con inyección dinámica de configuración y estado.

1.  **Trigger:** El usuario solicita "Crear Agente" via API/Dashboard.
2.  **Config Assembly:** El backend recopila la configuración (nombre, skills seleccionados, claves de API, límites de recursos).
3.  **Infrastructure Provisioning:** Se invoca la API de AWS ECS para instanciar una nueva tarea Fargate.
4.  **Runtime Bootstrap:** El contenedor arranca, monta su "cerebro" (EFS) y carga sus credenciales de forma segura.

---

## 2. Estrategia de Imagen Base

No construimos una imagen Docker nueva por cada agente. Utilizamos una imagen base optimizada y estandarizada para maximizar el caché y reducir tiempos de arranque.

*   **Imagen Oficial:** `openclaw/openclaw:latest` (o una versión semántica específica `v1.2.3` para producción).
*   **Composición:**
    *   **OS:** Alpine Linux o Debian Slim (minimizar superficie de ataque).
    *   **Runtime:** Node.js (LTS).
    *   **Core:** El binario/código fuente del agente OpenClaw pre-compilado.
    *   **Dependencias Base:** Herramientas comunes requeridas por skills estándar (curl, git, ffmpeg si se requiere voz/video).

### Inyección Dinámica de Skills y Plugins

Para evitar reconstruir imágenes cuando un usuario añade un plugin:

1.  **Plugins Core:** Los plugins más populares vienen pre-instalados en `/app/plugins` pero desactivados por defecto. Se activan via configuración.
2.  **Plugins Custom/Externos:**
    *   Al inicio (`entrypoint.sh`), el agente verifica una variable de entorno `PLUGIN_MANIFEST`.
    *   Si hay plugins externos, el agente los descarga/clona en un volumen efímero `/app/custom_plugins` antes de iniciar el proceso principal de Node.js.
    *   *Alternativa Performance:* Montar un volumen EFS compartido de "solo lectura" que contenga una biblioteca de todos los plugins versionados, montando solo lo necesario.

---

## 3. Orquestación con AWS Fargate

Utilizamos **AWS Fargate** para eliminar la gestión de servidores EC2 subyacentes, garantizando un aislamiento a nivel de kernel por defecto (cada tarea tiene su propia ENI y stack de red).

### Ciclo de Vida de la Task Definition

No creamos una "Task Definition" estática por cada agente. Usamos una **Task Definition Plantilla** y aplicamos "Overrides" al momento de ejecutar `RunTask`.

**Parámetros Sobrescritos en Runtime (Overrides):**
*   **CPU/Memoria:** Ajustado según el plan del cliente (ej. 0.5 vCPU / 1GB RAM vs 2 vCPU / 4GB RAM).
*   **Variables de Entorno:** Identificadores únicos del agente (`AGENT_ID`, `WORKSPACE_ID`).
*   **Tags:** Para facturación y control de costos (`CostCenter: ClientA`).

### Gestión Segura de Secretos (API Keys)

**NUNCA** pasamos API Keys como texto plano en variables de entorno estándar de la Task Definition (visibles en la consola AWS).

1.  **AWS Secrets Manager / Parameter Store:**
    *   Cuando el usuario guarda sus claves (OpenAI Key, Github Token), el backend las cifra y almacena en AWS Secrets Manager o SSM Parameter Store con una ruta específica: `/openclaw/agents/{agent_id}/secrets`.
2.  **Inyección en Contenedor:**
    *   En la definición del contenedor, usamos la propiedad `secrets`.
    *   ECS inyecta automáticamente el valor descifrado como una variable de entorno en el proceso del contenedor.
    *   El agente ve `OPENAI_API_KEY=sk-...` sin que el valor real toque el plano de control de ECS.
3.  **IAM Roles:**
    *   Cada tarea Fargate asume un **Task Execution Role** que tiene permisos estrictamente limitados (Least Privilege) para leer *solo* los secretos de ese agente específico (usando condiciones IAM por Resource Tag).

### Persistencia y Memoria (EFS)

El agente necesita "recordar" (archivos de memoria, logs, workspace). El almacenamiento efímero del contenedor se pierde al reiniciar.

1.  **Amazon EFS (Elastic File System):** Usamos un único sistema de archivos EFS masivo, segmentado por directorios.
2.  **Access Points (Puntos de Acceso):**
    *   Para cada Agente, se crea (o reutiliza) un **EFS Access Point**.
    *   Este Access Point fuerza al contenedor a operar dentro de un directorio específico: `/agents/{client_id}/{agent_id}`.
    *   También fuerza el `POSIX User ID` y `Group ID` con el que el contenedor escribe archivos.
3.  **Montaje:**
    *   El contenedor monta el EFS en `/home/openclaw/workspace`.
    *   Para el agente, parece un disco local vacío o con sus archivos previos.
    *   **Beneficio:** Si el contenedor muere y Fargate lo reinicia, se vuelve a montar el mismo directorio y el agente "despierta" con su memoria intacta.

---

## 4. Aislamiento y Seguridad (Multi-Tenant)

¿Cómo garantizamos que el Agente A no acceda a los datos del Agente B?

1.  **Aislamiento de Cómputo (Fargate):**
    *   Cada instancia de agente corre en su propia micro-VM gestionada por AWS (Firecracker microVM technology en el backend de Fargate). No comparten kernel ni memoria con otros agentes.
2.  **Aislamiento de Red (Security Groups):**
    *   **Egress:** Filtrado estricto. El agente solo puede salir a internet (HTTPS 443) para conectar con APIs de LLM y herramientas autorizadas.
    *   **Ingress:** CERRADO por defecto. No hay puertos abiertos escuchando. La comunicación es iniciada por el agente hacia el bus de mensajes (WebSocket/gRPC) o via Polling.
3.  **Aislamiento de Almacenamiento (EFS Access Points):**
    *   Es la capa crítica. Incluso si un atacante toma control del contenedor `root`, el EFS Access Point en el lado de AWS impide que el proceso lea o escriba fuera de su directorio asignado (`/agents/{su-id}/`). El sistema de archivos raíz del EFS es invisible para el contenedor.
4.  **Aislamiento de Identidad (IAM):**
    *   Las credenciales temporales de AWS inyectadas en el contenedor no tienen permisos para listar otros recursos, buckets S3 de otros clientes, o secretos ajenos.

---

## Resumen del Flujo de Arranque

1.  **Usuario:** Click "Start Agent".
2.  **Backend:**
    *   Genera credenciales temporales.
    *   Verifica existencia de directorio en EFS.
    *   Llama a `ecs:RunTask` con:
        *   Cluster: `openclaw-prod`
        *   TaskDef: `openclaw-agent-v1`
        *   Network: Subnet privada.
        *   Overrides: `env: AGENT_ID=xyz`, `secrets: [OPENAI_KEY_REF]`.
3.  **AWS Fargate:**
    *   Provisiona microVM.
    *   Pull imagen `openclaw/openclaw:latest`.
    *   Monta EFS Access Point `fsap-xyz` en `/workspace`.
    *   Inyecta secretos desencriptados en ENV.
4.  **Contenedor:**
    *   Inicia `node agent.js`.
    *   Lee `/workspace/memory.md` (persistente).
    *   Conecta al Gateway de OpenClaw.
    *   **Ready.**
