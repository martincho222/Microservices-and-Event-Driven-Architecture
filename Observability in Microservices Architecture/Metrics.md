# Metrics

---

## 1. What are Metrics

**Español:**
Las **métricas** son medidas numéricas del comportamiento de un sistema, recolectadas a intervalos regulares a lo largo del tiempo. A diferencia de los logs (que registran eventos individuales con todo su contexto) o las trazas (que siguen el recorrido de una request), las métricas **agregan** el comportamiento del sistema en series temporales de números. Esa agregación las hace extremadamente eficientes para almacenar y consultar, y las convierte en el mecanismo principal de **detección temprana de anomalías**, **alerting** y **análisis de tendencias** en un sistema de microservicios.

**English:**
**Metrics** are numerical measurements of system behavior, collected at regular intervals over time. Unlike logs (which record individual events with full context) or traces (which follow the journey of a request), metrics **aggregate** system behavior into time series of numbers. That aggregation makes them extremely efficient to store and query, and turns them into the primary mechanism for **early anomaly detection**, **alerting**, and **trend analysis** in a microservices system.

### Tipos de métricas

```
┌─────────────────────────────────────────────────────────────────┐
│           TIPOS DE MÉTRICAS — SHOPFAST                          │
├──────────────┬────────────────────────────────────┬────────────┤
│  Tipo        │  Descripción                       │  Ejemplo   │
├──────────────┼────────────────────────────────────┼────────────┤
│  Counter     │  Solo sube. Valor que se           │ Total de   │
│              │  incrementa con cada evento.        │ requests,  │
│              │  Nunca decrece (excepto al         │ total de   │
│              │  reiniciar el proceso).             │ errores    │
├──────────────┼────────────────────────────────────┼────────────┤
│  Gauge       │  Puede subir y bajar. Mide el      │ CPU usage, │
│              │  estado actual en un punto          │ memoria,   │
│              │  del tiempo.                        │ conexiones │
│              │                                    │ activas    │
├──────────────┼────────────────────────────────────┼────────────┤
│  Histogram   │  Distribuye valores en buckets      │ Latencia   │
│              │  predefinidos. Permite calcular     │ P50/P95/   │
│              │  percentiles (P50, P95, P99).       │ P99        │
├──────────────┼────────────────────────────────────┼────────────┤
│  Summary     │  Similar al histogram pero         │ Latencia   │
│              │  calcula percentiles en el          │ calculada  │
│              │  cliente. Menos flexible para       │ en el      │
│              │  agregación entre instancias.       │ servicio   │
└──────────────┴────────────────────────────────────┴────────────┘
```

### Stack tecnológico: Prometheus + Grafana

```
┌─────────────────────────────────────────────────────────────────┐
│         PROMETHEUS + GRAFANA — FLUJO EN SHOPFAST                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cada servicio expone /metrics (pull model, cada 15s):          │
│                                                                 │
│  http_requests_total{service="orders",status="200"} 48291       │
│  http_requests_total{service="orders",status="500"} 12          │
│  http_request_duration_seconds{quantile="0.99"}     0.048       │
│  jvm_memory_used_bytes{area="heap"}                 412058624   │
│                                                                 │
│  [Orders] [Payments] [Inventory] [Catalog] [Fraud]              │
│       │         │          │         │        │                 │
│       └─────────┴──────────┴─────────┴────────┘                │
│                            │                                    │
│                    [Prometheus]                                  │
│                    Scrape + time-series storage                  │
│                    Evalúa alerting rules (PromQL)                │
│                            │                                    │
│            ┌───────────────┴───────────────┐                    │
│       [Grafana]                   [Alertmanager]                 │
│       Dashboards y                Envía alertas a               │
│       visualización               PagerDuty / Slack             │
│       PromQL queries              cuando se supera umbral        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Metrics are your system's vital signs. Just like a doctor monitors heart rate and blood pressure — not every cell in your body — you monitor a small set of aggregated numbers, not every individual event. The art is choosing the right numbers to monitor."

---

## 2. The Problem of Collecting Too Many Metrics

**Español:**
Un error muy común en equipos que instrumentan microservicios por primera vez es **medir todo lo que pueden medir**. Cada variable, cada función, cada estado del sistema tiene una métrica. El resultado es una explosión de series temporales que satura Prometheus, genera dashboards ilegibles, produce decenas de alertas falsas simultáneas, y hace que el equipo deje de prestar atención a las alertas (alert fatigue). Paradójicamente, tener demasiadas métricas es casi tan malo como no tener ninguna: la señal se pierde entre el ruido, y durante un incidente real el equipo no sabe qué mirar.

**English:**
A very common mistake in teams instrumenting microservices for the first time is **measuring everything they can measure**. Every variable, every function, every system state has a metric. The result is an explosion of time series that saturates Prometheus, generates unreadable dashboards, produces dozens of simultaneous false alerts, and causes the team to stop paying attention to alerts (alert fatigue). Paradoxically, having too many metrics is almost as bad as having none: the signal is lost in the noise, and during a real incident the team does not know what to look at.

```
┌─────────────────────────────────────────────────────────────────┐
│         EL PROBLEMA DE LA CARDINALIDAD ALTA                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Un label con alta cardinalidad destruye Prometheus:            │
│                                                                 │
│  MAL — label userId (millones de valores únicos):               │
│  http_requests_total{service="orders", userId="usr-88"}  1      │
│  http_requests_total{service="orders", userId="usr-89"}  1      │
│  http_requests_total{service="orders", userId="usr-90"}  1      │
│  ... × 5.000.000 usuarios = 5M series temporales                │
│  → Prometheus se queda sin memoria. OOMKilled.                  │
│                                                                 │
│  BIEN — labels con cardinalidad baja y controlada:              │
│  http_requests_total{service="orders", status="200"}    48291   │
│  http_requests_total{service="orders", status="500"}    12      │
│  → Solo 2 series. Útil, eficiente, escalable.                   │
│                                                                 │
│  Regla de cardinalidad:                                         │
│  → Labels: solo valores de conjunto finito y pequeño            │
│    (status code, method, service name, environment)             │
│  → NUNCA: userId, orderId, sessionId, IP address como label     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Alert Fatigue — el síntoma de demasiadas métricas

```
┌─────────────────────────────────────────────────────────────────┐
│              ALERT FATIGUE — SHOPFAST                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Equipo con demasiadas métricas y alertas:                      │
│                                                                 │
│  Lunes 09:00 — 47 alertas activas                               │
│  Lunes 09:01 — ingeniero silencia todas                         │
│  Lunes 14:32 — incidente real: Payments caído                   │
│  Lunes 14:37 — nadie lo nota porque las alertas están           │
│               silenciadas o nadie las mira                      │
│  Lunes 15:10 — un usuario reporta que no puede pagar            │
│               38 minutos de downtime sin detección              │
│                                                                 │
│  La solución no es más alertas — es MEJORES alertas:            │
│  → Pocas alertas, todas accionables                             │
│  → Cada alerta tiene un runbook asociado                        │
│  → Si una alerta no requiere acción humana, no es una alerta    │
│  → Alertar sobre síntomas visibles al usuario, no sobre causas  │
│    internas (alerta sobre "error rate > 1%", no sobre           │
│    "CPU > 80%")                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### El costo real de demasiadas métricas

| Problema | Impacto |
|----------|---------|
| Alta cardinalidad | Prometheus OOMKilled, scrapes lentos, datos perdidos |
| Demasiadas series temporales | Costo de storage y query exponencialmente mayor |
| Dashboards sobrecargados | Nadie los mira; no hay señal clara durante incidentes |
| Alert fatigue | El equipo ignora alertas → incidentes reales sin detección |
| Esfuerzo de mantenimiento | Métricas desactualizadas que ya no se usan pero cuestan recursos |

> **Regla de Pogrebinsky:** "Don't instrument everything — instrument the right things. More metrics does not mean more observability. It means more noise. The goal is not to collect every possible number; the goal is to be able to answer 'is my system healthy?' and 'where is the problem?' as fast as possible."

---

## 3. The 5 "Golden Signals/Metrics"

**Español:**
Los **5 Golden Signals** (basados en los 4 Golden Signals del libro SRE de Google, con una extensión práctica) son el conjunto mínimo y suficiente de métricas que, juntas, responden la pregunta "¿está mi sistema saludable?" para cualquier microservicio. Son independientes del dominio de negocio: aplican igual a Orders Service que a Payments Service, a Catalog Service o a un API Gateway. Si solo puedes instrumentar una cosa en tu servicio, instrumenta estas cinco señales — con ellas tienes visibilidad operacional completa.

**English:**
The **5 Golden Signals** (based on Google's SRE book 4 Golden Signals, with a practical extension) are the minimum and sufficient set of metrics that together answer "is my system healthy?" for any microservice. They are domain-independent: they apply equally to Orders Service and Payments Service, to Catalog Service or an API Gateway. If you can only instrument one thing in your service, instrument these five signals — with them you have complete operational visibility.

```
┌─────────────────────────────────────────────────────────────────┐
│              LOS 5 GOLDEN SIGNALS — VISIÓN CONJUNTA             │
├────────────────────┬────────────────────────────────────────────┤
│  Signal            │  Pregunta que responde                     │
├────────────────────┼────────────────────────────────────────────┤
│  1. Latency        │  ¿Qué tan rápido responde el servicio?     │
│  2. Traffic        │  ¿Cuánta carga está recibiendo?            │
│  3. Errors         │  ¿Qué fracción de requests está fallando?  │
│  4. Saturation     │  ¿Qué tan lleno está el servicio?          │
│  5. Utilization    │  ¿Cuántos recursos está consumiendo?       │
└────────────────────┴────────────────────────────────────────────┘
```

---

### Signal 1 — Latency (Latencia)

**Español:**
La **latencia** mide el tiempo que tarda el servicio en responder a una request. No basta con medir el promedio — el promedio oculta los casos lentos que afectan a usuarios reales. El estándar es medir **percentiles**: P50 (la mediana), P95 y P99. El P99 indica que el 1% más lento de las requests tarda ese tiempo o más. En un sistema con 10,000 req/seg, el P99 afecta a 100 usuarios por segundo.

**English:**
**Latency** measures the time it takes the service to respond to a request. Measuring the average is not enough — the average hides slow cases that affect real users. The standard is to measure **percentiles**: P50 (the median), P95, and P99. The P99 indicates that the slowest 1% of requests take that long or more. In a system with 10,000 req/sec, the P99 affects 100 users per second.

```
┌─────────────────────────────────────────────────────────────────┐
│           LATENCY — SHOPFAST ORDERS SERVICE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Métricas clave:                                                │
│  http_request_duration_seconds{quantile="0.50"}  → P50: 18ms   │
│  http_request_duration_seconds{quantile="0.95"}  → P95: 85ms   │
│  http_request_duration_seconds{quantile="0.99"}  → P99: 210ms  │
│                                                                 │
│  Por qué el promedio engaña:                                    │
│  → 999 requests en 15ms + 1 request en 5000ms                  │
│  → Promedio: 19.98ms   → "Todo OK"                              │
│  → P99: 5000ms         → "El 1% espera 5 segundos"             │
│                                                                 │
│  Latencia de errores también importa:                           │
│  → Medir por separado: latencia de requests exitosas            │
│    vs latencia de requests fallidas                             │
│  → Una request que falla en 3s (timeout) es más dañina         │
│    que una que falla en 5ms (rechazo rápido)                   │
│                                                                 │
│  SLO típico de ShopFast checkout:                               │
│  P99 < 500ms para el 99.9% del tiempo en ventanas de 30 días   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Signal 2 — Traffic (Tráfico)

**Español:**
El **tráfico** mide la demanda que recibe el sistema: cuántas requests por segundo están llegando, cuántos mensajes se están consumiendo de Kafka, cuántas conexiones WebSocket están abiertas. El tráfico es el contexto imprescindible para interpretar los demás signals: un P99 de 200ms con 10 req/seg es muy diferente a un P99 de 200ms con 10,000 req/seg. El tráfico también detecta anomalías de negocio: una caída brusca del tráfico puede indicar un problema en el frontend o en el balanceador de carga.

**English:**
**Traffic** measures the demand the system is receiving: how many requests per second are arriving, how many messages are being consumed from Kafka, how many WebSocket connections are open. Traffic is the essential context for interpreting the other signals: a P99 of 200ms with 10 req/sec is very different from a P99 of 200ms with 10,000 req/sec. Traffic also detects business anomalies: a sudden drop in traffic can indicate a problem in the frontend or load balancer.

```
┌─────────────────────────────────────────────────────────────────┐
│           TRAFFIC — SHOPFAST                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Métricas clave por tipo de servicio:                           │
│                                                                 │
│  HTTP/REST services (Orders, Payments, Catalog):                │
│  → rate(http_requests_total[1m])  → requests/segundo            │
│                                                                 │
│  Event-driven consumers (Notifications, Shipping):              │
│  → kafka_consumer_records_consumed_total  → messages/segundo    │
│  → kafka_consumer_lag{group, topic}       → mensajes pendientes │
│                                                                 │
│  Patrones de tráfico que indican problemas:                     │
│  → Spike súbito: Black Friday, campaña de marketing             │
│    → necesita autoscaling + rate limiting                       │
│  → Caída a 0: el servicio dejó de recibir tráfico               │
│    → ¿el LB está roto? ¿el service discovery falló?            │
│  → Tráfico anormalmente alto: posible DDoS o bug de retry loop  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Examples:**
- `HTTP requests/sec` — throughput of synchronous REST/gRPC endpoints (Orders, Payments, Catalog)
- `Queries/sec` — database query rate; a spike indicates heavy read/write pressure on the DB
- `Transactions/sec` — completed business transactions end-to-end (checkouts, payments processed)
- `Events received/sec` — rate at which a consumer reads messages from a Kafka topic or SQS queue
- `Events delivered/sec` — rate at which a producer successfully publishes events to the broker

---

### Signal 3 — Errors (Errores)

**Español:**
Los **errores** miden la tasa de requests que fallan. Se dividen en errores **explícitos** (el servicio devuelve un 5xx) y errores **implícitos** (el servicio devuelve un 200 pero la respuesta es incorrecta — por ejemplo, un JSON vacío cuando debería tener datos). Los errores son el signal más directamente correlacionado con la experiencia del usuario. Un error rate del 1% en un servicio con 10,000 req/seg significa 100 usuarios fallando por segundo.

**English:**
**Errors** measure the rate of failing requests. They are divided into **explicit** errors (the service returns a 5xx) and **implicit** errors (the service returns a 200 but the response is incorrect — for example, an empty JSON when it should have data). Errors are the signal most directly correlated with user experience. A 1% error rate in a service with 10,000 req/sec means 100 users failing per second.

```
┌─────────────────────────────────────────────────────────────────┐
│           ERRORS — SHOPFAST                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Métricas clave:                                                │
│                                                                 │
│  Tasa de error HTTP:                                            │
│  rate(http_requests_total{status=~"5.."}[5m])                   │
│  ─────────────────────────────────────────────  × 100  = %err  │
│  rate(http_requests_total[5m])                                  │
│                                                                 │
│  Errores de negocio (implícitos):                               │
│  → orders_created_total{status="failed"}                        │
│  → payments_processed_total{result="declined"}                  │
│  → inventory_reservations_total{result="out_of_stock"}          │
│                                                                 │
│  Errores de dependencias externas:                              │
│  → db_query_errors_total{service="payments"}                    │
│  → kafka_producer_errors_total{topic="order.created"}           │
│                                                                 │
│  Alerta típica de ShopFast:                                     │
│  → error_rate > 1% durante 5 minutos → WARN → Slack             │
│  → error_rate > 5% durante 2 minutos → CRITICAL → PagerDuty    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Signal 4 — Saturation (Saturación)

**Español:**
La **saturación** mide qué tan "lleno" o "al límite" está el servicio en su recurso más crítico. A diferencia de la utilización (que mide consumo de recursos en general), la saturación mide específicamente la **cola de espera** y el **backpressure**: cuántas requests están esperando porque todos los workers están ocupados, cuántos mensajes de Kafka se acumulan sin procesar, cuántas conexiones están en cola esperando un slot del connection pool. La saturación es el signal que predice problemas futuros: cuando un servicio está saturado, la latencia sube y los errores están por llegar.

**English:**
**Saturation** measures how "full" or "at capacity" the service is at its most critical resource. Unlike utilization (which measures general resource consumption), saturation specifically measures the **wait queue** and **backpressure**: how many requests are waiting because all workers are busy, how many Kafka messages are accumulating unprocessed, how many connections are queued waiting for a connection pool slot. Saturation is the signal that predicts future problems: when a service is saturated, latency rises and errors are coming.

```
┌─────────────────────────────────────────────────────────────────┐
│           SATURATION — SHOPFAST                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread pool saturation (Orders Service):                       │
│  → tomcat_threads_busy / tomcat_threads_config_max              │
│    Si = 90%: las requests empiezan a hacer cola                 │
│    Si = 100%: las requests empiezan a rechazarse (503)          │
│                                                                 │
│  Connection pool saturation (Payments Service):                 │
│  → hikaricp_connections_active / hikaricp_connections_max       │
│    Si = 100%: nuevas queries esperan → latencia explota         │
│  (Fue exactamente este el incidente del incidente de ShopFast   │
│   descrito en TheThreePillarsOfObservability.md)                │
│                                                                 │
│  Kafka consumer lag (Notifications Service):                    │
│  → kafka_consumer_lag{group="notifications",topic="order.*"}    │
│    Si lag crece: el consumer no puede seguir el ritmo           │
│    del producer → mensajes se acumulan → notificaciones         │
│    tardías o nunca enviadas                                     │
│                                                                 │
│  Queue depth (procesamiento async):                             │
│  → jobs_queue_depth{service="shipping"}                         │
│    Si crece sostenidamente: el servicio está saturado            │
│                                                                 │
│  Regla clave: la saturación PREDICE problemas.                  │
│  Alertar cuando saturation > 80%, no cuando llega al 100%.      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Signal 5 — Utilization (Utilización de recursos)

**Español:**
La **utilización** mide qué fracción de los recursos disponibles (CPU, memoria, disco, red) está siendo consumida. Es el signal de **capacidad**: permite hacer capacity planning, detectar memory leaks (la memoria sube gradualmente sin volver a bajar) y anticipar si el servicio necesita más recursos antes de que el problema llegue a producción. La utilización no alerta por sí sola sobre problemas de usuario (un servicio puede estar al 90% de CPU y funcionar bien), pero combinada con latencia y error rate revela si la degradación viene de falta de recursos.

**English:**
**Utilization** measures what fraction of available resources (CPU, memory, disk, network) is being consumed. It is the **capacity** signal: it enables capacity planning, detects memory leaks (memory rises gradually without returning to baseline), and anticipates whether the service needs more resources before the problem reaches production. Utilization does not by itself alert about user-facing problems (a service can be at 90% CPU and work fine), but combined with latency and error rate it reveals whether degradation comes from resource shortage.

```
┌─────────────────────────────────────────────────────────────────┐
│           UTILIZATION — SHOPFAST                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CPU utilization:                                               │
│  → rate(process_cpu_seconds_total[1m]) × 100                    │
│  → Alerta si CPU > 80% sostenido: puede necesitar más réplicas  │
│                                                                 │
│  Memory utilization (detecta memory leaks):                     │
│  → jvm_memory_used_bytes{area="heap"}                           │
│  → Patrón de memory leak: la memoria sube con el tráfico        │
│    pero no baja en períodos de bajo tráfico                     │
│  → La GC corre cada vez más seguido → spikes de latencia        │
│                                                                 │
│  Disco (para servicios con estado):                             │
│  → node_filesystem_avail_bytes / node_filesystem_size_bytes     │
│  → Crítico para: Elasticsearch, Kafka brokers, PostgreSQL       │
│  → Disco lleno = servicio caído, a menudo sin posibilidad       │
│    de recuperación rápida                                       │
│                                                                 │
│  Network I/O:                                                   │
│  → node_network_transmit_bytes_total                            │
│  → Detecta: servicios que generan tráfico inesperado,           │
│    compresión insuficiente, payloads demasiado grandes          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Los 5 Golden Signals en conjunto — ShopFast Dashboard

```
┌─────────────────────────────────────────────────────────────────┐
│        DASHBOARD DE LOS 5 SIGNALS — PAYMENTS SERVICE           │
├────────────────┬──────────────────────┬────────┬───────────────┤
│  Signal        │  Métrica             │ Valor  │  Estado       │
├────────────────┼──────────────────────┼────────┼───────────────┤
│  Latency       │  P99 POST /charge    │ 4200ms │  ❌ CRÍTICO   │
│  Traffic       │  Requests/seg        │ 312    │  ✔ Normal     │
│  Errors        │  Error rate 5xx      │ 18%    │  ❌ CRÍTICO   │
│  Saturation    │  DB connection pool  │ 100%   │  ❌ SATURADO  │
│  Utilization   │  CPU                 │ 72%    │  ⚠ WARN       │
│                │  Memoria heap        │ 68%    │  ✔ Normal     │
├────────────────┴──────────────────────┴────────┴───────────────┤
│  Diagnóstico inmediato:                                         │
│  → Saturation: DB pool al 100% → cola de requests esperando    │
│  → Latency: P99 de 4.2s porque queries esperan conexión        │
│  → Errors: requests que timeout antes de obtener conexión      │
│  → Traffic: normal → el problema es interno, no un spike       │
│  → Acción: aumentar pool size + escalar réplicas de Payments   │
└─────────────────────────────────────────────────────────────────┘
```

### Beneficios y desventajas de los 5 Golden Signals

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS / LIMITACIONES    │
├──────────────────────────────┼──────────────────────────────────┤
│ Marco universal: aplica a    │ No cubren todo: bugs de lógica   │
│ cualquier servicio sin       │ de negocio (orden creada pero     │
│ importar el dominio o        │ sin cobrar) no se detectan con   │
│ tecnología.                  │ estos signals solos → necesitan  │
│                              │ métricas de negocio adicionales. │
│ Mínimo suficiente: con solo  │                                  │
│ 5 signals se puede           │ Latency y errors requieren       │
│ diagnosticar la mayoría de   │ instrumentación en el código     │
│ los incidentes en producción.│ (no son gratuitas). Saturation   │
│                              │ requiere conocer el recurso      │
│ Reducen el alert fatigue:    │ crítico de cada servicio.        │
│ en lugar de 50 alertas,      │                                  │
│ se tiene un conjunto pequeño │ No reemplazan las trazas ni los  │
│ y accionable.                │ logs: los signals dicen "hay     │
│                              │ un problema en Payments" pero    │
│ Estandarizan los dashboards  │ no dicen la causa raíz exacta    │
│ de todo el equipo: todos     │ → necesitas traces y logs para   │
│ saben qué buscar en un       │ completar el diagnóstico.        │
│ incidente.                   │                                  │
│                              │ Sampling de histogramas puede    │
│ Facilitan los SLOs: los      │ subestimar percentiles en        │
│ Service Level Objectives se  │ distribuciones con colas largas  │
│ definen naturalmente sobre   │ (tail latency).                  │
│ estos signals (P99 < 500ms,  │                                  │
│ error rate < 0.1%).          │                                  │
└──────────────────────────────┴──────────────────────────────────┘
```

### Resumen: los 5 Golden Signals de un vistazo

| Signal | Qué mide | Métrica Prometheus | Alerta típica |
|--------|----------|--------------------|---------------|
| **Latency** | Velocidad de respuesta (P50/P95/P99) | `histogram_quantile(0.99, http_request_duration_seconds_bucket)` | P99 > 500ms |
| **Traffic** | Carga recibida (req/s, msg/s) | `rate(http_requests_total[1m])` | Caída > 50% vs baseline |
| **Errors** | Tasa de fallos (%) | `rate(http_requests_total{status=~"5.."}[5m])` | Error rate > 1% |
| **Saturation** | Nivel de llenado del recurso crítico | `hikaricp_connections_active / hikaricp_connections_max` | > 80% de capacidad |
| **Utilization** | Consumo de recursos (CPU, memoria) | `rate(process_cpu_seconds_total[1m])` | CPU > 80% sostenido |

> **Regla de Pogrebinsky:** "If you instrument nothing else, instrument the 5 Golden Signals. They are the vital signs of your service. Latency tells you how sick it is. Traffic tells you how hard it's working. Errors tell you how many users are affected. Saturation tells you how close it is to collapse. Utilization tells you whether it has the resources to keep going."

