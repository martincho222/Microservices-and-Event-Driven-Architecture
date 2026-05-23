# Distributed Logging

---

## 1. What is Logging

**Español:**
El **logging** es el proceso de registrar eventos discretos que ocurren dentro de un sistema de software durante su ejecución. Un log es una línea de texto (o un objeto JSON estructurado) que describe qué pasó, cuándo pasó y en qué contexto. Los logs son la fuente de verdad más detallada sobre el comportamiento interno de un servicio: guardan el rastro exacto de cada operación, cada error, cada decisión de negocio y cada interacción con sistemas externos. A diferencia de las métricas (que agregan datos numéricos) o las trazas (que muestran el flujo de una request), los logs capturan el **evento en crudo** con todo su contexto.

**English:**
**Logging** is the process of recording discrete events that occur within a software system during its execution. A log is a line of text (or a structured JSON object) that describes what happened, when it happened, and in what context. Logs are the most detailed source of truth about a service's internal behavior: they keep the exact trail of every operation, every error, every business decision, and every interaction with external systems. Unlike metrics (which aggregate numerical data) or traces (which show the flow of a request), logs capture the **raw event** with all its context.

### Niveles de Log (Log Levels)

**Español:**
Los niveles de log permiten clasificar la severidad de cada evento y controlar qué se registra en cada entorno. Seguir una convención estricta de niveles es fundamental para no generar ruido innecesario (logs de DEBUG en producción que saturan el sistema) ni perder información crítica (errores reales mezclados con miles de líneas de INFO).

**English:**
Log levels allow classifying the severity of each event and controlling what is recorded in each environment. Following a strict level convention is essential to avoid generating unnecessary noise (DEBUG logs in production saturating the system) or losing critical information (real errors mixed with thousands of INFO lines).

```
┌─────────────────────────────────────────────────────────────────┐
│           NIVELES DE LOG — DE MENOR A MAYOR SEVERIDAD           │
├──────────┬────────────────────────────────────────┬────────────┤
│  Nivel   │  Cuándo usarlo                         │  Entorno   │
├──────────┼────────────────────────────────────────┼────────────┤
│  TRACE   │  Nivel más granular. Flujo interno      │  Local     │
│          │  del código paso a paso. Nunca en       │  dev only  │
│          │  producción.                            │            │
├──────────┼────────────────────────────────────────┼────────────┤
│  DEBUG   │  Información útil para desarrollo y     │  Dev / QA  │
│          │  diagnóstico detallado. Variables        │            │
│          │  intermedias, ramas condicionales.      │            │
├──────────┼────────────────────────────────────────┼────────────┤
│  INFO    │  Eventos de negocio normales y          │  Todos     │
│          │  esperados. "Orden creada", "Pago        │            │
│          │  procesado", "Servicio iniciado".        │            │
├──────────┼────────────────────────────────────────┼────────────┤
│  WARN    │  Situación inesperada pero recuperable. │  Todos     │
│          │  El sistema sigue funcionando.           │            │
│          │  "Retry 2/3 a Payments Service".        │            │
├──────────┼────────────────────────────────────────┼────────────┤
│  ERROR   │  Fallo que impide completar la          │  Todos     │
│          │  operación actual. Requiere atención.   │  (alertas) │
│          │  Stack trace incluido.                  │            │
├──────────┼────────────────────────────────────────┼────────────┤
│  FATAL   │  Error crítico que hace que el          │  Todos     │
│          │  proceso no pueda continuar             │  (pager)   │
│          │  ejecutándose. Previo a shutdown.       │            │
└──────────┴────────────────────────────────────────┴────────────┘

  Regla práctica de niveles en producción:
  → Nivel mínimo configurado: INFO
  → Solo se persisten: INFO + WARN + ERROR + FATAL
  → DEBUG/TRACE: se pueden activar on-demand por servicio
    sin redeploy (logging level dinámico)
```

### Logs estructurados vs no estructurados

```
┌─────────────────────────────────────────────────────────────────┐
│        LOGS NO ESTRUCTURADOS vs ESTRUCTURADOS — SHOPFAST        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  NO ESTRUCTURADO (texto libre):                                  │
│  ─────────────────────────────                                  │
│  2026-05-22 14:32:11 ERROR Failed to charge user 88 order 421   │
│  timeout after 3000ms calling payments-service                  │
│                                                                 │
│  Problemas:                                                      │
│  → No se puede filtrar por campo (ej: todos los errores de      │
│    un userId específico)                                        │
│  → No se puede agregar (ej: promedio de timeout durations)      │
│  → Frágil ante cambios de formato del mensaje                   │
│                                                                 │
│  ESTRUCTURADO (JSON):                                           │
│  ──────────────────                                             │
│  {                                                              │
│    "timestamp":  "2026-05-22T14:32:11.447Z",                    │
│    "level":      "ERROR",                                       │
│    "service":    "orders-service",                              │
│    "version":    "1.5.2",                                       │
│    "traceId":    "a3f7-bc12-9d44",                              │
│    "spanId":     "f1c3-aa08",                                   │
│    "userId":     "usr-88",                                      │
│    "orderId":    "421",                                         │
│    "message":    "Payment timeout",                             │
│    "durationMs": 3001,                                          │
│    "upstream":   "payments-service",                            │
│    "errorType":  "UpstreamTimeoutException"                     │
│  }                                                              │
│                                                                 │
│  Ventajas:                                                      │
│  → Filtrar: level=ERROR AND upstream=payments-service           │
│  → Agregar: AVG(durationMs) GROUP BY upstream                   │
│  → Correlacionar: todos los logs con traceId=a3f7-bc12-9d44    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "A log that a human can read but a machine cannot query is not a production log — it's a diary. In microservices, structured logs are not optional; they are the minimum requirement for centralized log management to work."

---

## 2. Distributed Logging — Best Practices

**Key practices:**
- **Centralized System** — aggregate logs from all services into a single searchable platform (ELK Stack, Grafana Loki, Datadog); pods are ephemeral and local logs are lost on restart
- **Predefined Structure / Schema** — enforce a consistent JSON schema across all services so every log has the same set of fields (timestamp, level, service, traceId, message); makes logs machine-queryable and dashboards reusable
- **Log Level / Log Severity** — assign the correct level (TRACE, DEBUG, INFO, WARN, ERROR, FATAL) to every log entry; configure production to emit INFO and above only; enable DEBUG dynamically per service without redeployment
- **Adding Contextual Information** — enrich every log with identifiers that make it actionable: `traceId`, `spanId`, `userId`, `orderId`, `service`, `version`, `environment`; a log without context is just a string

**Español:**
En una arquitectura de microservicios, el logging deja de ser un problema local de un servicio y se convierte en un problema distribuido. Una sola transacción de negocio (el checkout de ShopFast) genera logs en 6-10 servicios distintos, en 6-10 procesos distintos, en pods distintos de Kubernetes. Sin buenas prácticas, esos logs son un conjunto de fragmentos inconexos e inútiles para el diagnóstico. Las siguientes prácticas transforman ese conjunto fragmentado en una fuente cohesiva de verdad sobre el comportamiento del sistema.

**English:**
In a microservices architecture, logging is no longer a local problem for a single service — it becomes a distributed problem. A single business transaction (ShopFast checkout) generates logs in 6-10 different services, in 6-10 different processes, in different Kubernetes pods. Without good practices, those logs are a set of disconnected fragments useless for diagnosis. The following practices transform that fragmented set into a cohesive source of truth about system behavior.

---

### Best Practice 1 — Correlation ID (Trace ID)

**Español:**
El **Correlation ID** (también llamado **Trace ID** cuando se usa OpenTelemetry) es un identificador único que se genera al inicio de cada request y se propaga en todos los headers de las llamadas salientes. Cada servicio incluye ese ID en cada log que genera durante el procesamiento de esa request. Con esto, es posible buscar en el sistema centralizado de logs por ese ID y obtener **todos los logs de todos los servicios** que participaron en esa request, en orden cronológico.

**English:**
The **Correlation ID** (also called **Trace ID** when using OpenTelemetry) is a unique identifier generated at the start of each request and propagated in all outgoing call headers. Each service includes that ID in every log it generates while processing that request. This makes it possible to search the centralized log system by that ID and retrieve **all logs from all services** that participated in that request, in chronological order.

```
┌─────────────────────────────────────────────────────────────────┐
│           CORRELATION ID — FLUJO EN SHOPFAST                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. API Gateway genera: correlationId = "a3f7-bc12-9d44"        │
│     → Añade header: X-Correlation-ID: a3f7-bc12-9d44            │
│                                                                 │
│  2. Orders recibe la request:                                   │
│     → Lee X-Correlation-ID del header entrante                  │
│     → Incluye correlationId en TODOS sus logs                   │
│     → Propaga el header a Payments, Inventory, Fraud            │
│                                                                 │
│  3. Payments, Inventory, Fraud hacen lo mismo:                  │
│     → Leen el header → logean con ese ID → propagan             │
│                                                                 │
│  RESULTADO — búsqueda en Kibana por correlationId:              │
│                                                                 │
│  correlationId: "a3f7-bc12-9d44"                                │
│  ──────────────────────────────────────────────────             │
│  14:32:11.101 [api-gateway]   INFO  Request received POST /checkout
│  14:32:11.113 [orders]        INFO  Creating order #421         │
│  14:32:11.121 [inventory]     INFO  Reserving 2x SKU-889        │
│  14:32:11.216 [fraud-detect]  INFO  Fraud check passed (score:3) │
│  14:32:11.218 [orders]        INFO  Calling payments-service    │
│  14:32:14.219 [orders]        ERROR Payment timeout after 3001ms│
│  14:32:14.220 [orders]        INFO  Releasing inventory reserve │
│                                                                 │
│  → La historia completa del checkout fallido en 7 líneas        │
│    de 4 servicios distintos, perfectamente ordenadas            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Best Practice 2 — Centralized Log Aggregation

**Español:**
En producción, los microservicios corren en contenedores que pueden reiniciarse, reescalarse o moverse entre nodos de Kubernetes en cualquier momento. Si los logs se almacenan **localmente** en el contenedor, se pierden cuando el contenedor muere. La solución es un sistema de **agregación centralizada de logs**: un agente recolector corre en cada nodo, captura los logs de todos los contenedores y los envía a un almacén central indexado y buscable.

**English:**
In production, microservices run in containers that can restart, rescale, or move between Kubernetes nodes at any time. If logs are stored **locally** in the container, they are lost when the container dies. The solution is a **centralized log aggregation** system: a collector agent runs on each node, captures logs from all containers, and sends them to a central indexed and searchable store.

```
┌─────────────────────────────────────────────────────────────────┐
│           ARQUITECTURA DE LOG AGGREGATION — SHOPFAST            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  K8s Node 1                K8s Node 2                           │
│  ┌─────────────────┐       ┌─────────────────┐                  │
│  │ [orders-pod]    │       │ [payments-pod]  │                  │
│  │ [catalog-pod]   │       │ [inventory-pod] │                  │
│  │ stdout → JSON   │       │ stdout → JSON   │                  │
│  │       ↓         │       │       ↓         │                  │
│  │ [Filebeat/      │       │ [Filebeat/      │                  │
│  │  Fluent Bit]    │       │  Fluent Bit]    │                  │
│  │  DaemonSet      │       │  DaemonSet      │                  │
│  └────────┬────────┘       └────────┬────────┘                  │
│           │                         │                           │
│           └────────────┬────────────┘                           │
│                        ↓                                        │
│               [Logstash / Fluentd]                               │
│               Parsea, enriquece, filtra                          │
│               Añade metadata (env, cluster, region)              │
│                        ↓                                        │
│               [Elasticsearch]                                    │
│               Indexa y almacena los logs                         │
│               Retención: 30 días hot, 90 días warm               │
│                        ↓                                        │
│               [Kibana]                                           │
│               UI: buscar, dashboards, alertas                    │
│                                                                 │
│  Alternativas al ELK:                                           │
│  → Grafana Loki (más económico, indexa solo labels)             │
│  → Splunk (enterprise, muy potente, costoso)                    │
│  → Datadog Logs (SaaS, todo en uno con métricas y trazas)       │
│  → AWS CloudWatch Logs / GCP Cloud Logging                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Best Practice 3 — Log the Right Things (ni demasiado, ni poco)

**Español:**
Uno de los errores más comunes es loggear demasiado (cada línea del código, cada iteración de un bucle) o demasiado poco (solo los errores fatales). Demasiado volumen satura el sistema de logs, encarece el almacenamiento y hace imposible encontrar la señal entre el ruido. Muy poco volumen hace que los incidentes sean indiagnosticables. La guía es loggear los **eventos con significado de negocio o con utilidad para el diagnóstico**, no el flujo mecánico del código.

**English:**
One of the most common mistakes is logging too much (every line of code, every loop iteration) or too little (only fatal errors). Too much volume saturates the log system, increases storage costs, and makes it impossible to find the signal among the noise. Too little volume makes incidents undiagnosable. The guide is to log **events with business meaning or diagnostic utility**, not the mechanical flow of the code.

```
┌─────────────────────────────────────────────────────────────────┐
│         QUÉ LOGGEAR Y QUÉ NO — SHOPFAST ORDERS SERVICE          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✔ SÍ LOGGEAR:                                                  │
│  → Inicio y fin de cada request con duración total              │
│  → Toda excepción capturada (con stack trace en ERROR)          │
│  → Eventos de negocio: orden creada, pago procesado, envío      │
│    iniciado                                                     │
│  → Llamadas a servicios externos: inicio, resultado y duración  │
│  → Decisiones de negocio no triviales: "fraude detectado",      │
│    "cupón inválido", "stock insuficiente"                        │
│  → Retries: "retry 2/3 a inventory-service"                     │
│  → Inicio/shutdown del servicio con su versión                  │
│                                                                 │
│  ✗ NO LOGGEAR:                                                  │
│  → Cada iteración de un bucle que procesa colecciones           │
│  → Variables intermedias dentro de una función pura             │
│  → El contenido completo de objetos grandes en INFO             │
│    (solo IDs y campos clave)                                    │
│  → Operaciones de lectura exitosas de alta frecuencia           │
│    (GET /health/check 10.000 veces/min = ruido puro)            │
│  → Datos sensibles: contraseñas, tokens, números de tarjeta,    │
│    datos de salud (GDPR/PCI-DSS)                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Best Practice 4 — Sensitive Data Redaction (PII / PCI-DSS)

**Español:**
Los logs de producción **nunca deben contener datos sensibles**: contraseñas, tokens de sesión, números de tarjeta de crédito, datos de salud, ni información personal identificable (PII) como emails completos o documentos de identidad. En caso de breach de los logs, esos datos quedarían expuestos. El GDPR y PCI-DSS tienen requisitos explícitos sobre esto. La solución es **redactar o enmascarar** los campos sensibles antes de que lleguen al sistema de logs centralizado.

**English:**
Production logs **must never contain sensitive data**: passwords, session tokens, credit card numbers, health data, or personally identifiable information (PII) such as full emails or identity documents. In case of a log breach, that data would be exposed. GDPR and PCI-DSS have explicit requirements about this. The solution is to **redact or mask** sensitive fields before they reach the centralized log system.

```
┌─────────────────────────────────────────────────────────────────┐
│                REDACCIÓN DE DATOS SENSIBLES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MAL — log que incluye datos PCI:                               │
│  {                                                              │
│    "event": "payment_attempted",                                │
│    "cardNumber": "4111111111111111",   ← NUNCA                  │
│    "cvv": "123",                       ← NUNCA                  │
│    "cardholderName": "John Smith"      ← PII                    │
│  }                                                              │
│                                                                 │
│  BIEN — log con campos redactados / tokenizados:                │
│  {                                                              │
│    "event": "payment_attempted",                                │
│    "cardLast4": "1111",                ← solo los últimos 4     │
│    "cardToken": "tok_abc123",          ← token del gateway      │
│    "cardholderName": "J*** S***"       ← inicial + asteriscos   │
│  }                                                              │
│                                                                 │
│  Estrategias de redacción:                                      │
│  → En el código: nunca poner campos sensibles en el log         │
│  → En Logstash: mutate filter para enmascarar campos            │
│  → Auditoría periódica de los logs: revisar que no hay PII      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Best Practice 5 — Log Sampling para alto volumen

**Español:**
En servicios de muy alto tráfico (ej: Catalog Service con 2 millones de requests/día), loggear cada request en nivel INFO genera un volumen insostenible. La solución es el **log sampling**: para requests exitosas de alta frecuencia, se loggea solo una fracción (ej: 1 de cada 100). Los errores y eventos de negocio siempre se loggean al 100% — nunca se hace sampling sobre errores.

**English:**
In very high-traffic services (e.g., Catalog Service with 2 million requests/day), logging every request at INFO level generates an unsustainable volume. The solution is **log sampling**: for successful high-frequency requests, only a fraction is logged (e.g., 1 in 100). Errors and business events are always logged at 100% — sampling is never applied to errors.

```
  Catalog Service: GET /products/{id}
  ──────────────────────────────────────
  Status 200  →  loggear 1 de cada 100   (sampling 1%)
  Status 4xx  →  loggear 100%             (siempre)
  Status 5xx  →  loggear 100%             (siempre)
  Latencia > 1s → loggear 100%           (anomalía, siempre)

  Resultado: de 2M requests/día →
  logs reducidos a ~20K líneas de INFO + 100% de errores
```

---

### Best Practice 6 — Retention Policy y tiered storage

**Español:**
Los logs no deben almacenarse indefinidamente — el costo de storage crecería sin límite. La práctica estándar es definir una **política de retención por tier**: los logs recientes (hot) están en almacenamiento rápido y caro (SSD/Elasticsearch) para consultas en tiempo real durante incidentes. Los logs más antiguos (warm/cold) se mueven a almacenamiento más barato (S3, GCS) para cumplimiento y auditorías, pero no para búsquedas frecuentes.

**English:**
Logs should not be stored indefinitely — storage costs would grow without limit. The standard practice is to define a **tiered retention policy**: recent (hot) logs are in fast, expensive storage (SSD/Elasticsearch) for real-time queries during incidents. Older (warm/cold) logs move to cheaper storage (S3, GCS) for compliance and audits, but not for frequent searches.

```
┌─────────────────────────────────────────────────────────────────┐
│           POLÍTICA DE RETENCIÓN — SHOPFAST                      │
├──────────────┬───────────────────────┬─────────────────────────┤
│  Tier        │  Almacenamiento       │  Retención              │
├──────────────┼───────────────────────┼─────────────────────────┤
│  Hot         │  Elasticsearch (SSD)  │  7 días                 │
│              │  Búsqueda en < 1s     │  Para incidentes        │
├──────────────┼───────────────────────┼─────────────────────────┤
│  Warm        │  Elasticsearch (HDD)  │  30 días                │
│              │  Búsqueda en < 10s    │  Post-mortems           │
├──────────────┼───────────────────────┼─────────────────────────┤
│  Cold        │  S3 / GCS (comprimido)│  1 año (GDPR / PCI)     │
│              │  Solo descarga manual │  Solo compliance/legal  │
└──────────────┴───────────────────────┴─────────────────────────┘
```

---

### Resumen de Best Practices

| # | Best Practice | Problema que resuelve |
|---|---------------|-----------------------|
| 1 | **Correlation ID en todos los logs** | Imposible correlacionar logs de distintos servicios para una misma request |
| 2 | **Centralizar logs** (ELK / Loki) | Logs dispersos en pods efímeros que se pierden al reiniciar |
| 3 | **Loggear lo correcto** (ni ruido ni silencio) | Volumen excesivo que oculta señales reales, o logs insuficientes para diagnóstico |
| 4 | **Redactar datos sensibles** (PII / PCI) | Exposición de datos personales en caso de breach del sistema de logs |
| 5 | **Log sampling** para alto volumen | Costos de storage insostenibles en servicios de alto tráfico |
| 6 | **Retention policy** por tiers | Almacenamiento ilimitado y creciente, sin separación entre datos operacionales y de compliance |

> **Regla de Pogrebinsky:** "Distributed logging is not about having logs — every service has logs by default. It's about making those logs findable, correlatable, and actionable across dozens of services and thousands of pods. A log that nobody can find in time to fix an incident is the same as no log at all."