# Serverless Deployment for Microservices using Function as a Service

**Español:** El modelo **Serverless** representa un cambio de paradigma: en lugar de aprovisionar, configurar y mantener servidores (físicos o virtuales), el desarrollador se enfoca exclusivamente en escribir código de negocio. El proveedor de nube gestiona toda la infraestructura subyacente, incluyendo el sistema operativo, el runtime, el escalado y la alta disponibilidad.

**English:** The **Serverless** model represents a paradigm shift: instead of provisioning, configuring, and maintaining servers (physical or virtual), the developer focuses exclusively on writing business logic. The cloud provider manages all underlying infrastructure, including the operating system, runtime, scaling, and high availability.

> **Nota:** "Serverless" no significa que no hay servidores — significa que los servidores son invisibles para el desarrollador. Los servidores existen, pero son responsabilidad del proveedor de nube.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              EVOLUCIÓN DEL MODELO DE DESPLIEGUE                         │
│                                                                         │
│  On-Premises VM    →    Cloud VM        →    Containers    →  Serverless│
│                                                                         │
│  Tú gestionas:          Tú gestionas:        Tú gestionas:  Tú solo    │
│  • Hardware             • OS                 • App code     escribes:   │
│  • Network              • Runtime            • Dockerfile   • App code  │
│  • OS                   • App code           • Orchestrat.  • Config    │
│  • Runtime              • Scaling            • Scaling      (YAML)      │
│  • App code                                                             │
│  • Scaling              Proveedor:           Proveedor:     Proveedor:  │
│                         • Hardware           • Hardware     • Todo lo   │
│                         • Network            • Network        demás     │
│                                              • OS                       │
│                                              • Runtime                  │
│                                                                         │
│  Control máximo ◄──────────────────────────────────► Simplicidad máx.  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Motivation for Serverless Deployment using FaaS

**Español:** Antes de serverless, desplegar un microservicio — incluso uno pequeño — requería mantener una VM o un contenedor corriendo 24/7. Si ese servicio recibe 3 requests por hora, estás pagando por una instancia que está ociosa el 99.9% del tiempo. **Function as a Service (FaaS)** resuelve esto: el código se ejecuta solo cuando hay un evento que lo dispara, y el proveedor escala de 0 a miles de ejecuciones concurrentes automáticamente.

**English:** Before serverless, deploying a microservice — even a small one — required maintaining a VM or container running 24/7. If that service receives 3 requests per hour, you're paying for an instance that's idle 99.9% of the time. **Function as a Service (FaaS)** solves this: code executes only when an event triggers it, and the provider automatically scales from 0 to thousands of concurrent executions.

### El problema que resuelve / The problem it solves

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SIN SERVERLESS — SHOPFAST: notifications-service           │
│                                                                         │
│  Tráfico real del servicio de notificaciones (emails de confirmación):  │
│                                                                         │
│  00:00 ──────────────────── bajo ──────────────────── 08:00             │
│  08:00 ───── pico de mañana ───── 10:00                                 │
│  10:00 ──────────────────── bajo ──────────────────── 12:00             │
│  12:00 ─── pico del mediodía ─── 14:00                                  │
│  14:00 ──────────────────── bajo ──────────────────── 18:00             │
│  18:00 ──── pico de la tarde ─── 20:00                                  │
│  20:00 ──────────────────── bajo ──────────────────── 00:00             │
│                                                                         │
│  VM corriendo 24/7: t3.medium → $30/mes                                 │
│  Utilización promedio real: ~12%                                        │
│  Costo efectivo del trabajo real: ~$3.60/mes                            │
│  Desperdicio: ~88%                                                      │
│                                                                         │
│  CON SERVERLESS (Lambda):                                               │
│  Pago solo por ejecuciones reales → ~$1.20/mes (estimado 500K invocac.) │
│  Utilización: 100% (solo corre cuando hay trabajo)                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Qué es FaaS / What is FaaS

**Español:** **Function as a Service** es el modelo de ejecución que implementa serverless. El desarrollador sube una función (un bloque de código con una responsabilidad única), define el **trigger** (evento que la dispara) y el proveedor se encarga del resto. Cada invocación es stateless — la función no retiene estado entre llamadas.

**English:** **Function as a Service** is the execution model that implements serverless. The developer uploads a function (a block of code with a single responsibility), defines the **trigger** (event that fires it), and the provider handles the rest. Each invocation is stateless — the function retains no state between calls.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ANATOMÍA DE UNA FUNCIÓN FaaS                               │
│                                                                         │
│  ┌─────────────┐    trigger     ┌──────────────────────────────────┐    │
│  │   EVENT     │ ─────────────► │         FUNCTION                 │    │
│  │             │                │                                  │    │
│  │ • HTTP req  │                │  Runtime: Node.js / Python /     │    │
│  │ • Queue msg │                │           Java / Go / Ruby /     │    │
│  │ • Schedule  │                │           .NET / Custom          │    │
│  │ • DB change │                │                                  │    │
│  │ • File upload│               │  Memory:  128 MB – 10 GB         │    │
│  │ • IoT event │                │  Timeout: hasta 15 min (Lambda)  │    │
│  └─────────────┘                │  State:   STATELESS              │    │
│                                 └──────────────┬───────────────────┘    │
│                                                │ result                 │
│                                                ▼                        │
│                                  ┌─────────────────────┐               │
│                                  │  HTTP response /     │               │
│                                  │  Queue message /     │               │
│                                  │  DB write / etc.     │               │
│                                  └─────────────────────┘               │
│                                                                         │
│  Proveedor gestiona: instancias, OS, runtime, escalado, HA, parches     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Triggers más comunes / Most common triggers

| Trigger | Descripción | Ejemplo ShopFast |
|---|---|---|
| **HTTP / API Gateway** | Request HTTP/HTTPS entrante | `POST /orders` llama a `createOrder()` |
| **Message Queue** | Mensaje en SQS, SNS, Pub/Sub | `order.created` → `sendConfirmationEmail()` |
| **Schedule (Cron)** | Ejecución periódica | Cada noche: `generateDailyReport()` |
| **Storage event** | Archivo subido a S3 / GCS | Imagen subida → `resizeProductImage()` |
| **Database change** | DynamoDB Streams, Firestore | Cambio en inventario → `updateSearchIndex()` |
| **Stream** | Kafka, Kinesis, EventHub | Stream de eventos → `detectFraud()` |
| **IoT / WebSocket** | Evento de dispositivo | Sensor de almacén → `updateWarehouseStock()` |

### Modelo de escalado / Scaling model

```
┌─────────────────────────────────────────────────────────────────────────┐
│              ESCALADO AUTOMÁTICO — DE 0 A N EN SEGUNDOS                 │
│                                                                         │
│  Tráfico:  0 req/s    1 req/s   100 req/s  1000 req/s   0 req/s        │
│                                                                         │
│  VMs:      1 VM       1 VM       5 VMs      20 VMs       20 VMs        │
│            (idle)     (5% CPU)   (Auto      (escalando   (scale-in      │
│                                  Scaling,   ~5min)        ~10 min)      │
│                                  ~3 min)                                │
│                                                                         │
│  FaaS:     0 inst.    1 inst.    100 inst.  1000 inst.   0 inst.        │
│            (scale-0)  (cold      (caliente, (concurrente (scale-0       │
│                        start)    <1ms)      automático)  inmediato)     │
│                                                                         │
│  FaaS escala a 0 cuando no hay tráfico → $0 cuando no hay invocaciones  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## FaaS Deployment Benefits and Drawbacks

### Beneficios / Benefits

**Español:** FaaS ofrece ventajas significativas especialmente para servicios con tráfico variable, tareas event-driven y equipos pequeños que quieren minimizar el overhead operacional.

**English:** FaaS offers significant advantages especially for services with variable traffic, event-driven tasks, and small teams that want to minimize operational overhead.

#### 1. No Server Management

**Español:** El desarrollador no gestiona sistemas operativos, parches de seguridad, actualizaciones de runtime, configuración de red ni capacidad del servidor. El proveedor lo maneja todo. Un equipo de 2 personas puede operar decenas de funciones sin un DevOps dedicado.

**English:** The developer manages no operating systems, security patches, runtime updates, network configuration, or server capacity. The provider handles everything. A 2-person team can operate dozens of functions without a dedicated DevOps engineer.

#### 2. Automatic Scaling (incluyendo escala a cero / including scale to zero)

**Español:** La plataforma escala de forma automática e instantánea según la demanda, incluyendo escalar a cero instancias cuando no hay tráfico. No hay configuración de Auto Scaling Groups, Horizontal Pod Autoscalers ni políticas de escalado manual.

**English:** The platform scales automatically and instantly based on demand, including scaling to zero instances when there is no traffic. No Auto Scaling Group configuration, Horizontal Pod Autoscalers, or manual scaling policies are needed.

#### 3. Pay-per-Use Pricing

**Español:** Se paga únicamente por el tiempo de cómputo consumido, con granularidad de milisegundos. Si tu función no se invoca, no pagas nada. Modelo típico: $0.20 por millón de invocaciones + $0.0000166667 por GB-segundo.

**English:** You pay only for the compute time consumed, with millisecond granularity. If your function is not invoked, you pay nothing. Typical model: $0.20 per million invocations + $0.0000166667 per GB-second.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              COMPARACIÓN DE COSTOS — SHOPFAST notifications-service     │
│                                                                         │
│  Escenario: 2 millones de notificaciones/mes, 200ms promedio, 128MB     │
│                                                                         │
│  EC2 t3.medium (24/7):                                                  │
│    $30.37/mes (on-demand) — independiente del tráfico real              │
│                                                                         │
│  AWS Lambda:                                                            │
│    Invocaciones: 2M × $0.20/1M         = $0.40                          │
│    Cómputo:      2M × 0.2s × 128MB/1024 × $0.0000166667/GB-s           │
│                = 2M × 0.2 × 0.125 × $0.0000166667 = $0.83              │
│    Free tier:   -$0.40 (1M req) - $0.40 (400K GB-s)                    │
│    Total:        ~$0.83/mes                                             │
│                                                                         │
│  Ahorro: ~97% para este patrón de tráfico                               │
│  ⚠ Para servicios con tráfico constante y alto, EC2 puede ser más barato│
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4. Built-in High Availability

**Español:** Las funciones FaaS se ejecutan por defecto en múltiples zonas de disponibilidad. No hay que configurar load balancers, health checks ni replicación manual. Si una zona falla, el proveedor redirige automáticamente las invocaciones a otra zona.

**English:** FaaS functions run by default across multiple availability zones. No load balancers, health checks, or manual replication need to be configured. If a zone fails, the provider automatically redirects invocations to another zone.

#### 5. Faster Time to Market

**Español:** Sin infraestructura que gestionar, el equipo puede pasar de escribir código a tenerlo en producción en minutos. El ciclo completo de deploy es: escribir función → `zip` → subir → configurar trigger → listo.

**English:** With no infrastructure to manage, the team can go from writing code to having it in production in minutes. The complete deploy cycle is: write function → `zip` → upload → configure trigger → done.

#### 6. Event-Driven Architecture Natural Fit

**Español:** FaaS es el complemento natural de la arquitectura orientada a eventos. Cada evento del broker (Kafka, SNS, SQS) puede mapear directamente a una función, sin necesidad de un servicio persistente que consuma el topic. Los microservicios de procesamiento asíncrono son candidatos ideales.

**English:** FaaS is the natural complement to event-driven architecture. Each event from the broker (Kafka, SNS, SQS) can map directly to a function, without needing a persistent service consuming the topic. Asynchronous processing microservices are ideal candidates.

### Drawbacks / Inconvenientes

#### 1. Cold Start (Arranque en frío)

**Español:** Cuando una función no ha sido invocada recientemente, el proveedor necesita inicializar un nuevo contenedor de ejecución. Este proceso — llamado **cold start** — puede agregar entre 100ms y varios segundos de latencia. Es especialmente problemático para runtimes pesados como Java/JVM y para funciones que se llaman con poca frecuencia.

**English:** When a function has not been invoked recently, the provider needs to initialize a new execution container. This process — called a **cold start** — can add between 100ms and several seconds of latency. It is especially problematic for heavy runtimes like Java/JVM and for infrequently-called functions.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              COLD START vs WARM START                                   │
│                                                                         │
│  COLD START (primera invocación o después de inactividad):              │
│                                                                         │
│  [Download code] → [Start container] → [Init runtime] → [Execute]      │
│      ~50ms              ~150ms             ~200ms          ~50ms        │
│  Total: ~450ms adicionales                                              │
│                                                                         │
│  WARM START (invocación con instancia ya activa):                       │
│                                                                         │
│  [Execute]                                                              │
│    ~50ms                                                                │
│  Total: ~50ms                                                           │
│                                                                         │
│  Cold start por runtime (aproximado, Lambda):                           │
│  ┌──────────────┬──────────────┬──────────────────────────────────────┐ │
│  │ Runtime      │ Cold start   │ Mitigación                           │ │
│  ├──────────────┼──────────────┼──────────────────────────────────────┤ │
│  │ Python 3.x   │ 100-300ms    │ Uso por defecto                      │ │
│  │ Node.js 20   │ 100-300ms    │ Uso por defecto                      │ │
│  │ Go           │ 50-150ms     │ Mejor opción para baja latencia      │ │
│  │ Java 21      │ 1000-3000ms  │ GraalVM native image o Snapstart     │ │
│  │ .NET 8       │ 500-1500ms   │ ReadyToRun compilation               │ │
│  └──────────────┴──────────────┴──────────────────────────────────────┘ │
│                                                                         │
│  Soluciones: Provisioned Concurrency (Lambda), Minimum Instances,       │
│  keep-warm pings, lightweight runtimes (Go/Python/Node)                 │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 2. Execution Time Limits (Límites de tiempo de ejecución)

**Español:** Las funciones FaaS tienen un timeout máximo de ejecución. AWS Lambda permite hasta 15 minutos; Google Cloud Functions hasta 60 minutos; Azure Functions hasta 60 minutos en el plan Consumption. Esto las hace inadecuadas para procesos de larga duración como migraciones de datos, generación de reportes pesados o entrenamientos de modelos ML.

**English:** FaaS functions have a maximum execution timeout. AWS Lambda allows up to 15 minutes; Google Cloud Functions up to 60 minutes; Azure Functions up to 60 minutes on the Consumption plan. This makes them unsuitable for long-running processes like data migrations, heavy report generation, or ML model training.

#### 3. Stateless — Sin estado persistente entre invocaciones

**Español:** Cada invocación de una función es completamente independiente. No puedes almacenar estado en variables globales entre llamadas (el contenedor puede ser diferente). Para estado persistente necesitas servicios externos: bases de datos (DynamoDB, RDS), caché (ElastiCache/Redis) o almacenamiento de objetos (S3).

**English:** Each function invocation is completely independent. You cannot store state in global variables between calls (the container may be different). For persistent state you need external services: databases (DynamoDB, RDS), cache (ElastiCache/Redis), or object storage (S3).

#### 4. Vendor Lock-in

**Español:** Las funciones FaaS están profundamente integradas con el ecosistema del proveedor: triggers (SQS, SNS, EventBridge en AWS), bindings (Azure Functions), integraciones nativas (GCP Pub/Sub). Migrar de AWS Lambda a Google Cloud Functions requiere reescribir los handlers, triggers y configuraciones de red.

**English:** FaaS functions are deeply integrated with the provider's ecosystem: triggers (SQS, SNS, EventBridge on AWS), bindings (Azure Functions), native integrations (GCP Pub/Sub). Migrating from AWS Lambda to Google Cloud Functions requires rewriting handlers, triggers, and network configurations.

#### 5. Observability Complexity (Complejidad de observabilidad)

**Español:** Con VMs o contenedores tienes procesos de larga vida que puedes instrumentar continuamente. Con FaaS, las invocaciones son efímeras — cada ejecución genera su propio log stream, contexto de tracing, y métricas. Correlacionar logs de miles de invocaciones concurrentes requiere herramientas especializadas (AWS X-Ray, Datadog, Honeycomb).

**English:** With VMs or containers you have long-lived processes you can instrument continuously. With FaaS, invocations are ephemeral — each execution generates its own log stream, tracing context, and metrics. Correlating logs from thousands of concurrent invocations requires specialized tools (AWS X-Ray, Datadog, Honeycomb).

#### 6. Not Suitable for All Workloads

**Español:** FaaS no es la solución universal. Servicios con tráfico constante y alto, conexiones persistentes (WebSockets, gRPC streaming), o que necesitan mantener caché en memoria son malos candidatos. Para estos casos, contenedores o VMs son más apropiados.

**English:** FaaS is not a universal solution. Services with constant high traffic, persistent connections (WebSockets, gRPC streaming), or that need to maintain in-memory cache are poor candidates. For these cases, containers or VMs are more appropriate.

#### 7. If the Traffic Pattern Changes, Infrastructure Costs May Increase Significantly

**Español:** FaaS es económico para tráfico variable o bajo, pero si el patrón de tráfico cambia — por ejemplo, un servicio que pasa de 500K a 50 millones de invocaciones mensuales — el costo puede crecer de forma no lineal y superar ampliamente el costo de una VM o contenedor dedicado. La promesa de "pay-per-use" se convierte en "pay-per-every-request" a escala, y los equipos suelen ser tomados por sorpresa por facturas inesperadas.

**English:** FaaS is economical for variable or low traffic, but if the traffic pattern changes — for example, a service going from 500K to 50 million monthly invocations — the cost can grow non-linearly and far exceed the cost of a dedicated VM or container. The "pay-per-use" promise becomes "pay-per-every-request" at scale, and teams are often surprised by unexpected bills.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              COSTO FaaS vs CONTENEDOR A MEDIDA QUE CRECE EL TRÁFICO     │
│                                                                         │
│  Costo $                                                                │
│  │                                                    Lambda (/        │
│  │                                                   /                 │
│  │                                                  /                  │
│  │                                 Container ─────/──────              │
│  │                               ─────────────────                     │
│  │                         ──────                   \                  │
│  │                   ──────                          \                 │
│  │───────────────────                                 Lambda cheaper   │
│  └────────────────────────────────────────────────────────── Tráfico   │
│       bajo       medio        alto         muy alto                    │
│                                                                         │
│  Punto de quiebre (break-even) típico: ~2-5M invocaciones/mes          │
│  Por encima: evaluar migrar a contenedores o Reserved Instances.       │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 8. Unpredictable Performance

**Español:** El rendimiento de una función FaaS no es completamente controlable. Además del cold start, el entorno de ejecución comparte recursos de CPU con otras funciones del proveedor (modelo multi-tenant subyacente). Esto puede producir variabilidad en la latencia incluso entre invocaciones consecutivas de la misma función. Para SLOs con percentiles P99 muy estrictos (< 50ms), FaaS puede no ser la opción adecuada.

**English:** The performance of a FaaS function is not fully controllable. Beyond cold starts, the execution environment shares CPU resources with other provider functions (underlying multi-tenant model). This can produce latency variability even between consecutive invocations of the same function. For SLOs with very strict P99 percentiles (< 50ms), FaaS may not be the right choice.

#### 9. Multi-tenant Deployment

**Español:** A diferencia de un Dedicated Host, las funciones FaaS se ejecutan en infraestructura multi-tenant del proveedor. El código corre en contenedores aislados, pero el hardware físico subyacente es compartido con otros clientes. Para cargas de trabajo con requisitos estrictos de compliance (PCI-DSS nivel 1, HIPAA con BAA específico, FedRAMP High) o sensibles a ataques de canal lateral (side-channel attacks), este modelo puede ser inaceptable por política de seguridad.

**English:** Unlike a Dedicated Host, FaaS functions run on the provider's multi-tenant infrastructure. The code runs in isolated containers, but the underlying physical hardware is shared with other customers. For workloads with strict compliance requirements (PCI-DSS level 1, HIPAA with specific BAA, FedRAMP High) or sensitive to side-channel attacks, this model may be unacceptable per security policy.

### Tabla de Pros y Contras / Pros and Cons Table

| Dimensión | ✅ Beneficio | ❌ Inconveniente |
|---|---|---|
| **Gestión de infra** | Cero gestión de servidores, OS, parches | Menos control sobre el entorno de ejecución |
| **Escalado** | Automático, instantáneo, incluyendo escala a 0 | Límites de concurrencia por defecto (ajustables) |
| **Costo** | Pay-per-use; $0 cuando no hay tráfico | Costoso para cargas constantes y altas |
| **Latencia** | Warm start muy rápido | Cold start puede agregar 100ms–3s |
| **Estado** | Simplicidad: sin estado que gestionar | Requiere servicios externos para todo el estado |
| **Time to market** | Deploy en minutos, sin configurar infra | Depuración y testing local más complejo |
| **Event-driven** | Integración nativa con brokers de mensajes | Vendor lock-in en triggers y bindings |
| **Disponibilidad** | HA multi-AZ out of the box | Timeouts máximos; no apto para procesos largos |
| **Observabilidad** | Logs automáticos por invocación | Correlacionar miles de invocaciones es complejo |
| **Portabilidad** | — | Lock-in al proveedor (AWS/GCP/Azure) |

### Cuándo usar FaaS vs alternativas / When to use FaaS vs alternatives

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SHOPFAST — DECISIÓN POR SERVICIO                           │
│                                                                         │
│  ✅ IDEAL PARA FaaS:                                                    │
│  ────────────────────────────────────────────────────────────────────   │
│  notifications-service   → Lambda (event-driven, tráfico variable)      │
│  image-resize            → Lambda (trigger S3, ejecución corta)         │
│  fraud-detection-rules   → Lambda (trigger Kafka/Kinesis, stateless)    │
│  daily-reports           → Lambda (schedule/cron, 5 min ejecución)      │
│  webhook-receiver        → Lambda (HTTP trigger, burst de eventos)      │
│                                                                         │
│  ❌ MALA ELECCIÓN PARA FaaS:                                            │
│  ────────────────────────────────────────────────────────────────────   │
│  orders-service          → Container/VM (alto tráfico constante,        │
│                            conexiones DB persistentes, baja latencia)   │
│  search-service          → Container/VM (Elasticsearch en memoria,      │
│                            conexión persistente, warm cache crítico)    │
│  websocket-notifications → Container (conexión persistente WebSocket)   │
│  ml-training             → VM GPU (proceso >15 min, GPU requerida)      │
│                                                                         │
│  Regla: FaaS para lo event-driven y variable;                           │
│         Containers/VMs para lo constante y stateful.                    │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** Serverless no reemplaza los microservicios en contenedores — los complementa. La arquitectura óptima usa FaaS para los "glue services" (pegamento entre sistemas: notificaciones, procesamiento de eventos, transformaciones), y contenedores para los servicios core con tráfico predecible y alto.

### FaaS Providers — Principales plataformas

| Proveedor | Servicio | Runtime soportados | Timeout máx. | Límite memoria |
|---|---|---|---|---|
| **AWS** | Lambda | Node, Python, Java, Go, Ruby, .NET, Custom | 15 min | 10 GB |
| **Google Cloud** | Cloud Functions / Cloud Run Functions | Node, Python, Go, Java, Ruby, PHP, .NET | 60 min | 32 GB |
| **Microsoft Azure** | Azure Functions | C#, JavaScript, Python, Java, PowerShell | 60 min (Consumption) | 1.5 GB |
| **Cloudflare** | Workers | JavaScript / WASM | 30s (CPU) | 128 MB |
| **Vercel** | Edge Functions | JavaScript / TypeScript | 30s | 128 MB |
| **Open Source** | OpenFaaS / Knative | Cualquier lenguaje (Docker) | Configurable | Configurable |
---

## Serverless Deployment for Microservices using Function as a Service - Solutions

### Amazon Web Services (AWS)

**AWS Lambda** - A serverless, event-driven compute service that enables you to execute code for nearly any kind of application or backend service without the need to set up or oversee servers.

You can involve the execution of your code within a Lambda through more than 200 AWS services and various software-as-a-service (SaaS) applications.

You are billed solely for the resources you consume.

### Google Cloud Platform (GCP)

**Cloud Functions** - Allows executing your code in the cloud without the need to handle servers or containers. By utilizing GCP's scalable Functions as a Service (FaaS) offering, you only pay for what you use, following a pay-as-you-go model.

### Microsoft Azure

**Azure Functions** - An on-demand cloud service that provides the latest infrastructure and resources required to operate your applications. Allows you to fully focus on the code while Azure Functions manages the remaining aspects.
