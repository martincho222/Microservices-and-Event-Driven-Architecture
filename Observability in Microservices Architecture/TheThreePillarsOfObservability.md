# Introduction to the Three Pillars of Observability in Microservices

---

## 1. What is Observability

**Español:**
**Observabilidad** es la capacidad de entender el estado interno de un sistema a partir de sus salidas externas. El término viene de la teoría de control de ingeniería: un sistema es "observable" si, dada la observación de sus salidas durante un tiempo suficiente, se puede inferir completamente su estado interno. Aplicado a software, significa que cuando algo falla en producción, tienes suficiente información para entender **qué pasó, dónde pasó, por qué pasó y cuál fue el impacto** — sin necesidad de hacer SSH al servidor y correr comandos manualmente.

**English:**
**Observability** is the ability to understand the internal state of a system from its external outputs. The term comes from control engineering theory: a system is "observable" if, given observation of its outputs over sufficient time, its internal state can be fully inferred. Applied to software, it means that when something fails in production, you have enough information to understand **what happened, where it happened, why it happened, and what the impact was** — without needing to SSH into the server and run commands manually.

### Observabilidad vs Monitoreo — una distinción clave

```
┌─────────────────────────────────────────────────────────────────┐
│          MONITORING vs OBSERVABILITY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MONITORING (Monitoreo):                                        │
│  → Responde preguntas CONOCIDAS de antemano                     │
│  → "¿Está el servicio up?" / "¿Pasó el CPU del 80%?"           │
│  → Alertas configuradas sobre umbrales predefinidos             │
│  → Te dice CUÁNDO algo está mal                                 │
│  → Limitación: solo detecta fallos que alguien anticipó         │
│                                                                 │
│  OBSERVABILITY (Observabilidad):                                │
│  → Responde preguntas DESCONOCIDAS de antemano                  │
│  → "¿Por qué el P99 de checkout subió a 4s a las 14:32?"       │
│  → "¿Qué servicio específico de los 20 es el bottleneck?"      │
│  → Te dice POR QUÉ algo está mal y dónde buscar                 │
│  → Permite hacer preguntas arbitrarias sobre el sistema         │
│                                                                 │
│  Conclusión:                                                    │
│  Monitoring ⊂ Observability                                     │
│  El monitoreo es una herramienta dentro de la observabilidad,   │
│  no su equivalente.                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Monitoring tells you when your house is on fire. Observability tells you which room caught fire, why it started, and which other rooms are at risk — even for fires you've never seen before."

---

## 2. Importance of Observability

**Español:**
En un monolito, cuando algo falla, el desarrollador puede abrir el servidor, buscar en los logs de la aplicación y encontrar el stack trace. En una arquitectura de microservicios, una sola request de usuario puede atravesar 10, 15 o 20 servicios. Sin observabilidad, esa request es una caja negra: sabes que el usuario recibió un error, pero no sabes en qué servicio ocurrió, qué llamada lo desencadenó, ni si el problema es recurrente o aislado. La observabilidad transforma los microservicios de una caja negra en un sistema comprensible.

**English:**
In a monolith, when something fails, the developer can open the server, search the application logs, and find the stack trace. In a microservices architecture, a single user request may traverse 10, 15, or 20 services. Without observability, that request is a black box: you know the user received an error, but you don't know which service it occurred in, what call triggered it, or whether the problem is recurring or isolated. Observability transforms microservices from a black box into an understandable system.

```
┌─────────────────────────────────────────────────────────────────┐
│   SIN OBSERVABILIDAD — SHOPFAST CHECKOUT FALLIDO                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Usuario: "No puedo completar mi compra" ← incidente P1        │
│                                                                 │
│  La request atraviesa:                                          │
│  Browser → API Gateway → Orders → Inventory → Payments          │
│         → Fraud Detection → Shipping → Notifications            │
│                                                                 │
│  ¿Dónde falló?                                                  │
│  → No hay trazas distribuidas → no se sabe el camino recorrido  │
│  → Logs en 8 servicios distintos, en 8 servidores distintos     │
│  → No hay correlation ID → no se puede unir los logs            │
│  → El error en Orders dice: "upstream service timeout"          │
│    ¿Qué upstream? ¿Inventory? ¿Payments? ¿Fraud?               │
│                                                                 │
│  Resultado: el equipo pasa 2-3 horas en SSH y buscando logs     │
│  manualmente para encontrar algo que con observabilidad          │
│  hubiera tomado 5 minutos.                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│   CON OBSERVABILIDAD — EL MISMO INCIDENTE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Trace ID: a3f7-bc12-9d44                                       │
│                                                                 │
│  API Gateway   → Orders         [12ms]  ✔                       │
│  Orders        → Inventory      [8ms]   ✔                       │
│  Orders        → Fraud Detect.  [430ms] ✔ (lento pero OK)      │
│  Orders        → Payments       [3001ms] ✗ TIMEOUT              │
│                                                                 │
│  → En 5 segundos: el problema es Payments, específicamente      │
│    el endpoint POST /charge, que tardó 3 seg y time out         │
│                                                                 │
│  Métricas de Payments Service:                                  │
│  → Conexiones activas a DB: 98/100 (pool exhausto)             │
│  → Query "INSERT INTO charges" P99: 2800ms (baseline: 15ms)     │
│                                                                 │
│  Logs de Payments (correlacionados por trace ID):               │
│  [ERROR] HikariPool: Connection is not available, timeout 3000ms│
│                                                                 │
│  Diagnóstico: pool de conexiones DB agotado en Payments.        │
│  Tiempo total de diagnosis: 5 minutos.                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Por qué la observabilidad es más crítica en microservicios

| Característica | Monolito | Microservicios |
|----------------|----------|----------------|
| Nº de procesos a observar | 1 | 10 – 100+ |
| Stack trace completo | En un solo log | Distribuido en N servicios |
| Relación causa-efecto | Directa y local | Remota y encadenada (cascada) |
| Fallos de red | Raros (llamadas locales) | Frecuentes (HTTP / eventos) |
| Deploy independiente | No aplica | Necesitas saber qué versión falla |
| Necesidad de correlation | Baja | Crítica |

> **Regla de Pogrebinsky:** "In microservices, you don't debug — you observe. The system is too distributed and too dynamic to debug by SSHing into servers. Observability is not a luxury; it's the minimum viable requirement to operate a microservices system in production."

---

## 3. Three Pillars of Observability

**Español:**
Los **Tres Pilares de la Observabilidad** son los tres tipos de datos que, en conjunto, dan visión completa del estado de un sistema distribuido. Cada pilar responde a una pregunta diferente:
- **Logs** → ¿Qué ocurrió exactamente y en qué contexto?
- **Metrics** → ¿Cómo se está comportando el sistema cuantitativamente a lo largo del tiempo?
- **Distributed Traces** → ¿Cómo fluyó una request específica a través de todos los servicios?

**English:**
The **Three Pillars of Observability** are the three types of data that, together, give complete visibility into the state of a distributed system. Each pillar answers a different question:
- **Logs** → What exactly happened and in what context?
- **Metrics** → How is the system behaving quantitatively over time?
- **Distributed Traces** → How did a specific request flow through all the services?

```
┌─────────────────────────────────────────────────────────────────┐
│           LOS TRES PILARES — VISIÓN CONJUNTA                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   METRICS          LOGS              DISTRIBUTED TRACES         │
│  (Prometheus)     (ELK Stack)        (Jaeger / Zipkin)          │
│                                                                 │
│  "El P99 de       "El error fue      "La request del usuario   │
│  checkout subió   NullPointerExcept. fue: Gateway → Orders     │
│  a 4s a las       en OrderService    → Payments (TIMEOUT)      │
│  14:32"           línea 248"         → tardó 3001ms"           │
│                                                                 │
│  ¿Cuándo?         ¿Qué?              ¿Dónde y cómo?            │
│  ¿Cuánto?         ¿Por qué?          ¿Qué camino siguió?       │
│                                                                 │
│  Los tres pilares se complementan:                              │
│  Metrics detectan la anomalía → Traces identifican el servicio  │
│  → Logs explican la causa raíz                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Pilar 1 — Logs (Registros)

**Logs:**
- **Append-only files** — records are only added, never modified or deleted; preserves full history
- **Events in:**
  - **Application process** — business logic events, exceptions, request/response data, decisions
  - **Container** — lifecycle events (start, stop, crash, OOM kills), stdout/stderr of the process
  - **Database instance** — slow queries, connection errors, deadlocks, schema changes, audit events
  - **Server** — OS-level events, hardware errors, network issues, kernel messages
- **Structured / semi-structured strings** — typically JSON (structured) or key=value pairs (semi-structured), making them machine-readable and filterable
- **Metadata includes:**
  - **Timestamp** — when the event occurred (ISO 8601 / UTC)
  - **Request** — request ID or trace ID to correlate across services
  - **Method** — HTTP method, function name, or operation type
  - **Class** — the class or module that generated the log entry
  - **Application** — the service or component name (e.g., `orders-service`)

**Español:**
Los **logs** son registros textuales de eventos discretos que ocurren en el sistema: una request recibida, una excepción lanzada, una query ejecutada, una decisión de negocio tomada. En microservicios, el desafío no es generar logs — todos los frameworks lo hacen por defecto — sino **centralizar** los logs de todos los servicios en un único lugar con capacidad de búsqueda y correlación. El estándar actual es usar **logs estructurados en JSON** (en lugar de texto libre) para que sean machine-readable y fácilmente filtrables.

**English:**
**Logs** are textual records of discrete events that occur in the system: a received request, a thrown exception, an executed query, a business decision made. In microservices, the challenge is not generating logs — all frameworks do it by default — but **centralizing** logs from all services in a single place with search and correlation capabilities. The current standard is to use **structured logs in JSON** (instead of free text) so they are machine-readable and easily filterable.

```
┌─────────────────────────────────────────────────────────────────┐
│       LOGS ESTRUCTURADOS vs LOGS DE TEXTO — SHOPFAST            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LOG DE TEXTO (malo para búsqueda):                             │
│  2026-05-22 14:32:11 ERROR OrderService - Failed to process     │
│  order for user 88, payment timeout after 3000ms                │
│                                                                 │
│  LOG ESTRUCTURADO EN JSON (machine-readable):                   │
│  {                                                              │
│    "timestamp":  "2026-05-22T14:32:11.447Z",                    │
│    "level":      "ERROR",                                       │
│    "service":    "orders-service",                              │
│    "version":    "1.5.2",                                       │
│    "traceId":    "a3f7-bc12-9d44",    ← correlation ID          │
│    "spanId":     "f1c3-aa08",                                   │
│    "userId":     "usr-88",                                      │
│    "orderId":    "421",                                         │
│    "message":    "Payment timeout",                             │
│    "durationMs": 3001,                                          │
│    "errorType":  "UpstreamTimeoutException",                    │
│    "upstream":   "payments-service"                             │
│  }                                                              │
│                                                                 │
│  → Permite buscar: todos los errores del usuario usr-88         │
│  → Permite correlacionar: todos los logs con traceId a3f7-...  │
│  → Permite agregar: promedio de durationMs por servicio          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Stack tecnológico de Logs: ELK Stack

```
┌─────────────────────────────────────────────────────────────────┐
│               ELK STACK — FLUJO EN SHOPFAST                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Orders]  [Payments]  [Catalog]  [Shipping]  [Inventory]       │
│     │           │          │          │           │             │
│     └───────────┴──────────┴──────────┴───────────┘            │
│                           │                                     │
│                    [Logstash / Filebeat]                         │
│                    Recolector de logs                            │
│                    Parsea, enriquece y transforma                │
│                           │                                     │
│                    [Elasticsearch]                               │
│                    Motor de búsqueda y almacenamiento            │
│                    Indexa los logs JSON                          │
│                    Retención configurable (30/90 días)           │
│                           │                                     │
│                    [Kibana]                                      │
│                    UI de búsqueda y dashboards                   │
│                    Queries en KQL / Lucene                       │
│                    Alertas sobre patrones de logs                │
│                                                                 │
│  Alternativas al ELK:                                           │
│  → Grafana Loki + Promtail (más ligero, indexa solo labels)     │
│  → Splunk (enterprise)                                          │
│  → AWS CloudWatch Logs / GCP Cloud Logging (cloud-native)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Beneficios y desventajas — Logs

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Máximo nivel de detalle:     │ Alto volumen de datos: ShopFast  │
│ el log contiene el contexto  │ puede generar GB/hora de logs,   │
│ exacto del error (stack       │ lo que eleva el costo de         │
│ trace, parámetros, IDs).     │ almacenamiento y procesamiento.  │
│                              │                                  │
│ Diagnóstico de causa raíz:   │ Costo de query alto: Elasticsearch│
│ cuando la métrica dice        │ con grandes volúmenes es lento   │
│ "algo está mal", el log      │ para queries ad-hoc en tiempo    │
│ dice exactamente qué.        │ real durante un incidente.       │
│                              │                                  │
│ Auditoría y compliance:      │ Logs no estructurados son        │
│ trazabilidad de quién hizo   │ inútiles a escala: texto libre   │
│ qué, cuándo y con qué datos. │ no se puede agregar ni filtrar   │
│                              │ eficientemente.                  │
│ Correlación con traceId:     │                                  │
│ al incluir el trace ID en    │ Riesgo de incluir PII (datos     │
│ cada log, se pueden unir     │ personales) en logs → problema   │
│ todos los logs de una        │ de GDPR/compliance si no se      │
│ request distribuida.         │ anonimiza correctamente.         │
└──────────────────────────────┴──────────────────────────────────┘
```

---

### Pilar 2 — Metrics (Métricas)

**Metrics:**
- **Regularly sampled data points** — collected at fixed intervals (e.g., every 15s by Prometheus scrape)
- **Numerical values:**
  - **Counters** — monotonically increasing values that only go up (e.g., total requests, total errors)
  - **Distribution** — statistical spread of values over a time window; used for latency histograms and percentiles (P50, P95, P99)
  - **Gauges** — point-in-time values that can go up or down (e.g., current memory usage, active connections)
- **Examples:**
  - `Requests/min` — throughput rate per service or endpoint
  - `Errors/hour` — count of 4xx/5xx responses over time
  - `Latency distribution` — P50/P95/P99 of response times
  - `Current CPU utilization` — gauge: % of CPU in use right now
  - `Memory usage` — gauge: heap / RSS memory consumed by the process
  - `Cache hit rate` — ratio of cache hits vs total lookups (e.g., Redis in Catalog Service)

**Español:**
Las **métricas** son medidas numéricas agregadas del comportamiento del sistema a lo largo del tiempo: cuántas requests por segundo procesa un servicio, cuál es la latencia promedio y el percentil 99, cuánto CPU y memoria consume, cuántos errores ocurren por minuto. A diferencia de los logs (que capturan eventos individuales), las métricas **agregan** el comportamiento en números que se pueden graficar, comparar y sobre los que se pueden configurar alertas. Son el mecanismo de **detección temprana** de problemas.

**English:**
**Metrics** are aggregated numerical measurements of system behavior over time: how many requests per second a service processes, what the average latency and 99th percentile are, how much CPU and memory it consumes, how many errors occur per minute. Unlike logs (which capture individual events), metrics **aggregate** behavior into numbers that can be graphed, compared, and used to configure alerts. They are the **early detection** mechanism for problems.

### Tipos de métricas en microservicios

```
┌─────────────────────────────────────────────────────────────────┐
│            TIPOS DE MÉTRICAS — SHOPFAST                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. MÉTRICAS DE INFRAESTRUCTURA (nivel sistema):                │
│     CPU usage, memory usage, disk I/O, network I/O              │
│     Fuente: el agente de la VM/container (Node Exporter)        │
│                                                                 │
│  2. MÉTRICAS DE APLICACIÓN — RED (Rate / Errors / Duration):   │
│     Patrón "RED" — el estándar para microservicios HTTP/gRPC:   │
│     Rate:     requests/segundo por endpoint                     │
│     Errors:   tasa de errores 5xx (%) por endpoint              │
│     Duration: latencia P50 / P95 / P99 por endpoint             │
│                                                                 │
│  3. MÉTRICAS DE NEGOCIO (Business Metrics):                     │
│     órdenes_creadas_total{status="success"}                     │
│     pagos_procesados_total{método="card"}                       │
│     revenue_total{moneda="USD"}                                 │
│     carritos_abandonados_total                                  │
│     → Detectan que algo está mal antes que las métricas técnicas│
│       ej: error rate OK pero órdenes completadas cayeron 30%    │
│                                                                 │
│  4. MÉTRICAS DE DEPENDENCIAS EXTERNAS:                          │
│     db_connection_pool_active{service="payments"}               │
│     kafka_consumer_lag{group="payments", topic="order.created"} │
│     redis_hit_rate{cache="catalog"}                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Stack tecnológico de Métricas: Prometheus + Grafana

```
┌─────────────────────────────────────────────────────────────────┐
│         PROMETHEUS + GRAFANA — FLUJO EN SHOPFAST                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cada servicio expone un endpoint /metrics en formato           │
│  Prometheus (text-based, pull model):                           │
│                                                                 │
│  # HELP http_requests_total Total HTTP requests                 │
│  # TYPE http_requests_total counter                             │
│  http_requests_total{method="POST",status="200"} 4382           │
│  http_requests_total{method="POST",status="500"} 7              │
│  http_request_duration_seconds{quantile="0.99"} 0.045           │
│                                                                 │
│  [Orders /metrics] [Payments /metrics] [Catalog /metrics]       │
│         │                  │                  │                 │
│         └──────────────────┴──────────────────┘                │
│                            │                                    │
│                     [Prometheus]                                 │
│                     Scrape cada 15s los /metrics                 │
│                     Almacena time-series                         │
│                     Evalúa alerting rules                        │
│                            │                                    │
│               ┌────────────┴────────────┐                       │
│          [Grafana]               [Alertmanager]                  │
│          Dashboards               Envía alertas a                │
│          PromQL queries           PagerDuty / Slack              │
│          Visualización            cuando se superan umbrales     │
│                                                                 │
│  Alternativas:                                                   │
│  → Datadog (SaaS, todo en uno)                                  │
│  → AWS CloudWatch Metrics / Azure Monitor                       │
│  → InfluxDB + Telegraf                                          │
│  → OpenTelemetry Collector (vendor-neutral, recomendado)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Beneficios y desventajas — Metrics

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Detección temprana: alertas  │ Bajo nivel de detalle: una       │
│ en tiempo real cuando un     │ métrica dice "el P99 subió" pero │
│ umbral se supera (error rate │ no dice por qué ni en qué        │
│ > 1%, P99 > 500ms).          │ request específica ocurrió.      │
│                              │                                  │
│ Bajo costo de almacenamiento:│ Cardinalidad alta es cara:       │
│ un counter es un número —    │ labels con muchos valores únicos  │
│ millones de requests se      │ (ej: userId) pueden explotar     │
│ reducen a una serie temporal.│ el almacenamiento de Prometheus. │
│                              │                                  │
│ Dashboards y tendencias:     │ Pull model puede perder métricas │
│ se puede ver la evolución    │ de procesos de vida corta        │
│ del sistema a lo largo del   │ (jobs) que terminan antes del    │
│ tiempo (semanas, meses).     │ siguiente scrape.                │
│                              │                                  │
│ Alerting preciso: Alertmanager│ Solo miden lo que se instrumenta│
│ + PromQL permite reglas de   │ explícitamente — los problemas   │
│ alerta muy específicas       │ que nadie anticipó no tienen     │
│ (ej: tasa de error por       │ métricas configuradas.           │
│ endpoint y por servicio).    │                                  │
└──────────────────────────────┴──────────────────────────────────┘
```

---

### Pilar 3 — Distributed Traces (Trazas Distribuidas)

**Español:**
Las **trazas distribuidas** (distributed traces) son el registro del recorrido completo de una request a través de todos los microservicios. Una traza está compuesta por un árbol de **spans**: cada span representa una operación unitaria (una llamada HTTP, una query a DB, una publicación de mensaje a Kafka), con su duración, su servicio de origen, su servicio de destino y sus metadatos. Todos los spans de una misma request comparten un **Trace ID** único, lo que permite reconstruir el árbol completo aunque la request haya pasado por 15 servicios.

**English:**
**Distributed traces** are the complete journey record of a request through all microservices. A trace is composed of a tree of **spans**: each span represents a unit operation (an HTTP call, a DB query, a Kafka message publication), with its duration, source service, destination service, and metadata. All spans from the same request share a unique **Trace ID**, which allows reconstructing the complete tree even if the request passed through 15 services.

```
┌─────────────────────────────────────────────────────────────────┐
│       ESTRUCTURA DE UNA TRAZA — SHOPFAST CHECKOUT               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Trace ID: a3f7-bc12-9d44   Total: 3248ms                       │
│                                                                 │
│  [API Gateway]      ├──────────────────────────────────┤  12ms  │
│    [Orders]         │  ├──────────────────────────────┤  3230ms │
│      [DB Query]     │  │  ├──────┤                       8ms    │
│      [Inventory]    │  │  │  ├──────────────┤            95ms   │
│        [DB Query]   │  │  │  │  ├──┤                    12ms   │
│      [Fraud Detect] │  │  │  ├──────────────────┤        430ms  │
│        [ML model]   │  │  │  │  ├────────────┤           380ms  │
│      [Payments]     │  │  │  ├──────────────────────────┤ 3001ms│
│        [DB Query]   │  │  │  │  │  (timeout, no response)│      │
│      [Payments]     │  │  │  │  ERROR: connection timeout│      │
│                                                                 │
│  → Un solo vistazo muestra: Payments tardó 3001ms               │
│  → La DB de Payments no respondió dentro del timeout            │
│  → Los demás servicios son rápidos y correctos                  │
│                                                                 │
│  Propagación del Trace ID (W3C Trace Context estándar):         │
│  → Orders → Payments: Header "traceparent: 00-a3f7bc12...-01"  │
│  → Payments → DB driver: el SDK lo propaga automáticamente      │
│  → Cada servicio crea su span hijo con parentSpanId             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Stack tecnológico de Trazas: Jaeger y Zipkin

```
┌─────────────────────────────────────────────────────────────────┐
│     JAEGER / ZIPKIN — FLUJO EN SHOPFAST                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cada servicio instrumentado con OpenTelemetry SDK:             │
│  → Crea spans automáticamente para HTTP in/out                  │
│  → Crea spans para queries SQL, llamadas a Kafka                │
│  → Propaga el Trace ID en los headers de salida                 │
│                                                                 │
│  [Orders] [Payments] [Inventory] [Fraud] [Catalog] [Shipping]   │
│      │         │          │        │         │         │        │
│      └─────────┴──────────┴────────┴─────────┴─────────┘       │
│                           │                                     │
│               [OpenTelemetry Collector]                          │
│               Recibe, procesa y exporta spans                    │
│               Compatible con múltiples backends                  │
│                           │                                     │
│               ┌───────────┴───────────┐                         │
│           [Jaeger]               [Zipkin]                        │
│           Backend de trazas      Backend de trazas               │
│           UI para buscar         (alternativa, más simple)       │
│           trazas por:            Elasticsearch / Cassandra       │
│           - Trace ID             como backend de storage         │
│           - Servicio                                             │
│           - Operación            Alternativas comerciales:       │
│           - Duración > X ms      → Datadog APM                  │
│           - Estado de error      → New Relic                     │
│                                  → AWS X-Ray                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Sampling — el problema del volumen de trazas

**Español:**
ShopFast procesa 10,000 requests/segundo en hora punta. Almacenar una traza completa por cada request sería 10,000 trazas/seg × 30 spans promedio = 300,000 spans/seg — prohibitivamente caro. La solución es el **sampling**: capturar solo un porcentaje de las trazas. Hay dos estrategias principales:

**English:**
ShopFast processes 10,000 requests/second at peak hour. Storing a complete trace per request would be 10,000 traces/sec × 30 avg spans = 300,000 spans/sec — prohibitively expensive. The solution is **sampling**: capturing only a percentage of traces. There are two main strategies:

```
┌─────────────────────────────────────────────────────────────────┐
│              SAMPLING STRATEGIES                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HEAD-BASED SAMPLING (simple):                                  │
│  → El API Gateway decide al inicio de la request si la traza    │
│    se captura: "capturar 1 de cada 100" (1%)                    │
│  → Problema: no sabe de antemano si la request fallará          │
│  → El 1% de trazas de error también se descartan → se pierden   │
│    exactamente las trazas más valiosas para debugging           │
│                                                                 │
│  TAIL-BASED SAMPLING (inteligente):                             │
│  → Captura TODAS las trazas inicialmente                        │
│  → Cuando la traza completa llega al collector, evalúa:         │
│    ¿Tuvo algún span con error? → GUARDAR (100% de errores)     │
│    ¿Tardó más de 1 segundo?   → GUARDAR (slow requests)        │
│    ¿Request normal?            → DESCARTAR 99%                  │
│  → Resultado: alta fidelidad en los casos que importan         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Beneficios y desventajas — Distributed Traces

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Visibilidad end-to-end: el   │ Overhead de instrumentación:     │
│ recorrido completo de una    │ crear y propagar spans añade      │
│ request en todos los         │ latencia (~1-5ms) y CPU por       │
│ servicios en una sola vista. │ request — aceptable pero no cero. │
│                              │                                  │
│ Identificación de            │ Sampling hace perder trazas:     │
│ bottlenecks: qué servicio    │ con head-based sampling, los     │
│ o qué operación específica   │ bugs esporádicos pueden no       │
│ es el cuello de botella.     │ capturarse.                      │
│                              │                                  │
│ Depuración de errores        │ Requiere instrumentar todos los  │
│ distribuidos: correlacionar  │ servicios — si un servicio       │
│ el error en Payments con     │ viejo no está instrumentado,     │
│ la request original del      │ la traza tiene un hueco.         │
│ usuario en el browser.       │                                  │
│                              │                                  │
│ Análisis de dependencias:    │ Alto costo de storage: trazas    │
│ el flame graph de trazas     │ completas son voluminosas —      │
│ muestra qué servicios llaman │ Elasticsearch o Cassandra como   │
│ a qué otros (service map).   │ backend + retención limitada.    │
└──────────────────────────────┴──────────────────────────────────┘
```

---

### Los Tres Pilares en conjunto — flujo de un incidente real

```
┌─────────────────────────────────────────────────────────────────┐
│       WORKFLOW DE OBSERVABILIDAD — INCIDENTE EN SHOPFAST        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  14:32 — ALERTA (Metrics):                                      │
│  Grafana/Alertmanager → PagerDuty:                              │
│  "orders_success_rate cayó de 99.9% a 96.8%                     │
│   y P99 de checkout subió a 4.2s"                               │
│  → El equipo sabe: HAY UN PROBLEMA en el flujo de checkout      │
│                                                                 │
│  14:33 — DIAGNÓSTICO (Traces):                                  │
│  Ingeniero abre Jaeger → filtra por: error=true, last 5min      │
│  → Ve que el 100% de las trazas con error muestran:             │
│    Payments span: 3001ms (TIMEOUT)                              │
│  → El problema está en Payments Service, endpoint POST /charge  │
│                                                                 │
│  14:34 — CAUSA RAÍZ (Logs):                                     │
│  Ingeniero va a Kibana → filtra: service=payments, level=ERROR  │
│  → En 30 segundos encuentra:                                    │
│  [ERROR] HikariPool-1: Timeout acquiring connection.            │
│  Pool stats: active=100, idle=0, waiting=47                     │
│  → El connection pool de la DB de Payments está agotado         │
│                                                                 │
│  14:37 — RESOLUCIÓN:                                            │
│  → Aumentar pool size de 100 a 200 (hotfix de config)           │
│  → Verificar con Metrics: P99 vuelve a 45ms en 2 minutos        │
│  → Total tiempo de resolución: 5 minutos                        │
│                                                                 │
│  SIN OBSERVABILIDAD: el mismo incidente = 2-3 horas de SSH      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Tabla resumen de los tres pilares

| Dimensión | Logs | Metrics | Distributed Traces |
|-----------|------|---------|-------------------|
| ¿Qué responde? | ¿Qué ocurrió exactamente? | ¿Cuánto / qué tan rápido? | ¿Cómo fluyó la request? |
| Granularidad | Evento individual | Agregado en el tiempo | Request individual |
| Uso primario | Debugging de causa raíz | Alerting y tendencias | Diagnóstico de latencia |
| Costo de storage | Alto (GB/hora) | Bajo (series numéricas) | Medio-alto (spans) |
| Herramientas JVM | Logback + ELK | Micrometer + Prometheus | OpenTelemetry + Jaeger |
| Herramientas Cloud | CloudWatch Logs | CloudWatch Metrics | AWS X-Ray |
| Tiempo de búsqueda | Medio (query text) | Rápido (series numéricas) | Rápido (por Trace ID) |
| Requiere sampling | No | No aplica | Sí (tail-based recomendado) |

> **Regla de Pogrebinsky:** "You need all three pillars — not one, not two. Metrics without traces tell you something is slow but not where. Traces without logs tell you which service failed but not why. Logs without metrics tell you what happened but you'll find out hours later. Together, they reduce your Mean Time To Resolution from hours to minutes."