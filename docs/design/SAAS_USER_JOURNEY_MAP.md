# OpenClaw SaaS User Journey Map

## Objetivo del Documento
Este documento detalla el viaje completo del usuario en la plataforma SaaS de OpenClaw, desde el descubrimiento hasta la operación continua de sus agentes. El objetivo es proporcionar transparencia total sobre la interacción del usuario y los procesos del sistema, ocultando la complejidad de la infraestructura subyacente (AWS Fargate) mientras se resalta la experiencia fluida de gestión de agentes.

## Diagrama Global del Journey (Mermaid)

```mermaid
graph TD
    %% Estilos
    classDef userAction fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef systemProcess fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef criticalStep fill:#fff9c4,stroke:#fbc02d,stroke-width:2px,stroke-dasharray: 5 5;
    
    %% Fases
    subgraph F1 [Fase 1: Adquisición y Descubrimiento]
        A1(Usuario busca soluciones de Agentes AI) --> A2(Aterriza en Landing Page OpenClaw)
        A2 --> A3{¿Interesado?}
        A3 -- No --> A4(Abandono / Retargeting)
        A3 -- Sí --> A5[Clic en 'Comenzar Gratis' / 'Crear Cuenta']
    end

    subgraph F2 [Fase 2: Onboarding y Registro]
        B1(Formulario de Registro: Email/Pass o SSO):::userAction --> B2(Sistema envía Email de Verificación):::systemProcess
        B2 --> B3(Usuario hace clic en enlace de verificación):::userAction
        B3 --> B4{¿Email Validado?}:::systemProcess
        B4 -- No --> B5(Mostrar error / Reenviar)
        B4 -- Sí --> B6(Redirección al Dashboard Inicial):::systemProcess
        B6 --> B7(Usuario ve Dashboard en modo 'Solo Lectura' o 'Limitado'):::userAction
    end

    subgraph F3 [Fase 3: Compra de Créditos - CRÍTICO]
        C1(Usuario intenta crear Agente o accede a Billing):::userAction --> C2(Sistema solicita Saldo):::systemProcess
        C2 --> C3(Usuario selecciona Paquete de Créditos):::criticalStep
        C3 --> C4(Procesamiento de Pago - Stripe/Provider):::systemProcess
        C4 --> C5{¿Pago Exitoso?}
        C5 -- No --> C6(Notificación de fallo / Reintentar)
        C5 -- Sí --> C7(Asignación de Créditos a la Cuenta):::systemProcess
        C7 --> C8(Desbloqueo de funciones de creación):::systemProcess
    end

    subgraph F4 [Fase 4: Configuración del Agente]
        D1(Usuario clic en 'Nuevo Agente'):::userAction --> D2(Selección de Imagen Base):::userAction
        D2 -.-> D2a[Opciones: OpenClaw Standard / Custom Docker]
        D2 --> D3(Configuración de Recursos):::userAction
        D3 -.-> D3a[Slider: vCPU / RAM]
        D3 --> D4(Inyección de Secretos):::userAction
        D4 -.-> D4a[API Keys: OpenAI, Anthropic, etc.]
        D4 --> D5(Definición de Persistencia / Workspace):::userAction
        D5 --> D6(Revisión de Costo Estimado por Hora):::systemProcess
        D6 --> D7(Clic en 'Desplegar Agente'):::userAction
    end

    subgraph F5 [Fase 5: Operación y Ejecución (Transparente)]
        E1(Sistema valida Créditos Disponibles):::systemProcess --> E2(Provisionamiento AWS Fargate - Invisible):::systemProcess
        E2 --> E3(Pull de Imagen Docker):::systemProcess
        E3 --> E4(Inyección de Variables de Entorno Seguras):::systemProcess
        E4 --> E5(Inicio del Contenedor / Agente):::systemProcess
        E5 --> E6(Agente Reporta Estado 'Ready'):::systemProcess
        E6 --> E7(Usuario ve Estado 'Online' en Dashboard):::userAction
    end

    subgraph F6 [Fase 6: Interacción y Consumo]
        F1_Act(Usuario abre Terminal/Interfaz de Control):::userAction --> F2_Act(Conexión WebSocket Segura):::systemProcess
        F2_Act --> F3_Act(Envío de Comandos / Tareas):::userAction
        F3_Act --> F4_Act(Agente Ejecuta y Responde):::systemProcess
        F4_Act --> F5_Act(Medición de Uso en Tiempo Real):::systemProcess
        F5_Act --> F6_Act(Deducción de Créditos del Saldo):::criticalStep
        F6_Act --> F7_Act{¿Saldo Agotado?}
        F7_Act -- Sí --> F8_Act(Suspensión/Alerta al Usuario)
        F7_Act -- No --> F9_Act(Continuar Operación)
    end

    %% Conexiones entre fases
    A5 --> B1
    B7 --> C1
    C8 --> D1
    D7 --> E1
    E7 --> F1_Act
```

## Detalle Paso a Paso

### 1. Adquisición
El usuario llega principalmente por tres canales:
- **Búsqueda Orgánica (SEO):** Buscando "hosting de agentes AI", "alternativa a ejecutar agentes locales", "OpenClaw cloud".
- **Referral/Social:** Enlaces directos a configuraciones de agentes compartidos o repositorios de GitHub.
- **Directo:** Usuarios de la versión open-source que buscan escalar a la nube.

**Punto de contacto:** Landing Page optimizada que destaca: "Despliega tu agente OpenClaw en segundos, sin gestionar servidores".

### 2. Onboarding
El proceso está diseñado para reducir la fricción inicial pero asegurar la identidad.
1.  **Registro:** El usuario introduce correo/contraseña o usa OAuth (GitHub/Google).
2.  **Validación:** Se envía un correo con un token de un solo uso.
3.  **Primer Acceso:** Al hacer clic, el usuario entra al "Command Center".
    *   *Estado:* La cuenta es "Unverified" o "Free Tier" (si aplica), con capacidad de ver pero no de desplegar cargas de trabajo pesadas.

### 3. Compra de Créditos (The Paywall)
Este es el momento de la verdad. OpenClaw SaaS funciona bajo un modelo de **prepago/consumo**.
1.  **Trigger:** El usuario intenta crear su primer agente o accede proactivamente a la sección de "Billing".
2.  **Interfaz de Compra:**
    *   El usuario ve paquetes claros: "$10 (Starter)", "$50 (Pro)", etc.
    *   Se muestra la equivalencia estimada: "$10 equivale a aprox. 100 horas de agente estándar".
3.  **Transacción:** Procesamiento seguro (Stripe).
4.  **Confirmación:** El saldo se actualiza instantáneamente en la UI (header superior).
    *   *Feedback:* "¡Créditos añadidos! Ahora puedes desplegar tu primer agente."

### 4. Configuración del Agente (The Builder)
El usuario configura la "máquina" sin saber que es un contenedor Fargate.
1.  **Nombre y Rol:** Asigna un nombre amigable (ej: "DevBot-Alpha") y una descripción.
2.  **Imagen Base (El Cerebro):**
    *   Selecciona `openclaw/std:latest` (recomendado) o introduce su propia imagen Docker desde ECR/DockerHub público.
3.  **Dimensionamiento (El Cuerpo):**
    *   El usuario mueve un slider o selecciona presets:
        *   *Micro:* 0.5 vCPU / 1GB RAM (Para tareas ligeras).
        *   *Standard:* 1 vCPU / 2GB RAM.
        *   *Power:* 2 vCPU / 4GB RAM.
    *   *Calculadora en tiempo real:* Al cambiar la selección, se actualiza el "Costo por hora estimado" en créditos.
4.  **Secretos y Variables (Las Llaves):**
    *   Interfaz segura para introducir `OPENAI_API_KEY`, `GITHUB_TOKEN`, etc.
    *   Estos valores se inyectan como variables de entorno encriptadas en el arranque.
5.  **Persistencia:** Opción para adjuntar un volumen persistente (EFS) si el agente necesita recordar datos entre reinicios (costo adicional mostrado).

### 5. Operación (The Deployment)
Lo que sucede tras hacer clic en "Desplegar":
1.  **Validación de Saldo:** El sistema verifica si hay saldo suficiente para al menos 1 hora de ejecución.
2.  **Orquestación (Caja Negra):**
    *   El backend de OpenClaw solicita a AWS Fargate una tarea con las especificaciones definidas.
    *   Se configura la red (VPC, Security Groups) automáticamente para aislar al agente.
3.  **Bootstrapping:**
    *   El agente inicia.
    *   Se conecta al "Gateway" de OpenClaw mediante WebSocket saliente (evitando necesidad de IPs públicas de entrada complejas para el usuario).
4.  **Ready:**
    *   En el Dashboard del usuario, el indicador de estado pasa de `PROVISIONING` (Amarillo) a `ONLINE` (Verde).
    *   Tiempo estimado: 30-60 segundos.

### 6. Interfaz de Control
El usuario interactúa con su agente vivo.
1.  **Terminal Web:** Una consola en el navegador conectada directamente al STDIN/STDOUT del agente.
2.  **Visor de Archivos:** Explorador para ver el `workspace` del agente en tiempo real.
3.  **Logs:** Pestaña para ver logs de sistema y depuración.
4.  **Botón de Pánico:** "Stop Agent" detiene el contenedor inmediatamente y frena el consumo de créditos.

### 7. Ciclo de Facturación (Consumo)
- El sistema mide el tiempo de ejecución por minuto.
- Cada minuto, se descuenta la fracción correspondiente de créditos del saldo del usuario.
- **Alertas:**
    - Al 20% de saldo restante: Email + Notificación en Dashboard.
    - Al 0%: El agente se pausa/hiberna (estado `SUSPENDED`) y se notifica al usuario para recargar.
