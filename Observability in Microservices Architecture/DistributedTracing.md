# Distributed Tracing

---

## 1. Distributed Tracing — Motivation

**Español:**
En una arquitectura de microservicios, una sola acción del usuario puede desencadenar decenas de llamadas entre servicios. Cuando algo falla o va lento, los logs y las métricas dicen *qué* está fallando, pero no *por qué camino llegó la request hasta ese punto* ni *cómo se relacionan los eventos de distintos servicios*. Sin trazabilidad distribuida, debuggear un problema significa abrir los logs de 10-15 servicios por separado, intentar correlacionarlos manualmente por timestamp, y rezar porque los relojes de los servidores estén sincronizados. El **Distributed Tracing** resuelve esto registrando el recorrido completo de cada request como un objeto único y coherente que abarca todos los servicios involucrados.

**English:**
In a microservices architecture, a single user action can trigger dozens of calls between services. When something fails or is slow, logs and metrics say *what* is failing, but not *what path the request took to get there* or *how events from different services relate to each other*. Without distributed tracing, debugging a problem means opening logs from 10-15 services separately, trying to correlate them manually by timestamp, and hoping the server clocks are synchronized. **Distributed Tracing** solves this by recording the complete journey of each request as a single coherent object spanning all involved services.

```
┌─────────────────────────────────────────────────────────────────┐
│       SIN DISTRIBUTED TRACING — SHOPFAST CHECKOUT LENTO         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Usuario: "El checkout me tarda 8 segundos" ← SLA: < 1s        │
│                                                                 │
│  Request atraviesa:                                             │
│  Browser → API Gateway → Orders → Inventory → Payments          │
│         → Fraud Detection → Shipping Estimator → Response       │
│                                                                 │
│  Ingeniero sin tracing:                                         │
│  1. SSH a servidor de Orders → revisar logs → "llamó a 3 deps"  │
│  2. SSH a servidor de Payments → logs → "respondió en 200ms"   │
│  3. SSH a servidor de Fraud → logs → "respondió en 420ms"      │
│  4. SSH a servidor de Shipping → logs → "respondió en 7100ms"  │
│     → ¿es Shipping? ¿o algo que Shipping llama?                │
│  5. SSH a servidor de... (30 minutos después)                   │
│                                                                 │
│  Ingeniero CON tracing:                                         │
│  1. Abrir Jaeger UI                                             │
│  2. Buscar: service=api-gateway, duration > 5s                  │
│  3. Ver el flame graph en 5 segundos:                           │
│     → Shipping Estimator llamó a Maps API externa              │
│       y tardó 7100ms esperando respuesta (sin timeout!)         │
│  Tiempo total de diagnóstico: 2 minutos                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Por qué el logging centralizado no es suficiente

```
┌─────────────────────────────────────────────────────────────────┐
│       LOGS vs TRACES — LO QUE CADA UNO RESPONDE                 │
├────────────────────────┬────────────────────────────────────────┤
│  Logs centralizados    │  Distributed Traces                    │
├────────────────────────┼────────────────────────────────────────┤
│ "El pago falló con     │ "La request tardó 8s en total porque  │
│ NullPointerException   │ Orders llamó a Inventory (90ms) →     │
│ en línea 248"          │ luego a Fraud (420ms) → luego a        │
│                        │ Shipping que llamó a Maps API (7100ms)"│
├────────────────────────┼────────────────────────────────────────┤
│ ¿Qué pasó?             │ ¿Cómo fluyó la request?               │
│ ¿Por qué falló?        │ ¿Qué servicio fue el cuello de botella?│
│ Contexto del error     │ ¿Cuánto tardó cada paso?              │
│                        │ ¿Qué llamadas se hicieron en paralelo? │
├────────────────────────┼────────────────────────────────────────┤
│ Nivel: evento          │ Nivel: request end-to-end              │
│ individual             │ a través de N servicios                │
└────────────────────────┴────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Logs tell you what happened inside a service. Distributed traces tell you what happened across your entire system for a specific user request. You need both — traces to find where the problem is, logs to understand why it happened."

---

## 2. Terminology + How Distributed Tracing Works

### Terminología fundamental

```
┌─────────────────────────────────────────────────────────────────┐
│              TERMINOLOGÍA DE DISTRIBUTED TRACING                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TRACE                                                          │
│  ─────                                                          │
│  El recorrido completo de una request a través de todos los     │
│  servicios que participaron en su procesamiento.                │
│  Identificado por un único Trace ID (UUID).                     │
│  Una traza es un árbol de spans.                                │
│                                                                 │
│  SPAN                                                           │
│  ────                                                           │
│  Una operación unitaria dentro de la traza: una llamada HTTP,   │
│  una query a la DB, una publicación a Kafka, una llamada a una  │
│  función. Un span tiene:                                        │
│  → spanId:       identificador único de este span               │
│  → parentSpanId: el span que lo originó (null en el root span)  │
│  → traceId:      el trace al que pertenece                      │
│  → operationName: "POST /orders", "SELECT payments", etc.       │
│  → startTime + duration: cuándo empezó y cuánto tardó           │
│  → tags/attributes: metadata clave-valor (userId, orderId...)   │
│  → status: OK / ERROR                                           │
│                                                                 │
│  ROOT SPAN                                                      │
│  ─────────                                                      │
│  El primer span de una traza. No tiene parentSpanId.            │
│  Generalmente lo crea el API Gateway o el primer servicio        │
│  que recibe la request del exterior.                            │
│                                                                 │
│  CONTEXT PROPAGATION                                            │
│  ────────────────────                                           │
│  El mecanismo por el que el traceId y el spanId se transmiten   │
│  de un servicio al siguiente, para que cada servicio sepa a     │
│  qué traza pertenece su trabajo y quién es su span padre.       │
│  Estándar: W3C Trace Context (header "traceparent")             │
│                                                                 │
│  INSTRUMENTATION                                                │
│  ───────────────                                                │
│  El código (librería o agente) que crea y reporta spans         │
│  automáticamente. Con OpenTelemetry, la mayoría de frameworks   │
│  (Spring Boot, Express, Django) se auto-instrumentan.           │
│                                                                 │
│  SAMPLING                                                       │
│  ────────                                                       │
│  No todas las trazas se almacenan — solo una fracción.          │
│  Head-based: la decisión se toma al inicio de la request.       │
│  Tail-based: la decisión se toma cuando la traza completa       │
│  llega al collector (puede guardar 100% de los errores).        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Anatomía de una traza — ShopFast checkout

```
┌─────────────────────────────────────────────────────────────────┐
│   TRAZA COMPLETA — Trace ID: a3f7-bc12-9d44   Total: 8241ms     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Span 1 [root]: API Gateway  POST /checkout         8241ms      │
│  │  spanId: s1  parentSpanId: null                              │
│  │                                                              │
│  ├── Span 2: Orders Service  POST /orders           8229ms      │
│  │   │  spanId: s2  parentSpanId: s1                            │
│  │   │                                                          │
│  │   ├── Span 3: DB Query  INSERT orders            8ms  ✔      │
│  │   │   spanId: s3  parentSpanId: s2                           │
│  │   │                                                          │
│  │   ├── Span 4: Inventory  POST /reserve           95ms  ✔     │
│  │   │   spanId: s4  parentSpanId: s2                           │
│  │   │   └── Span 5: DB Query  UPDATE inventory    11ms  ✔      │
│  │   │                                                          │
│  │   ├── Span 6: Fraud Detect.  POST /check        420ms  ✔     │
│  │   │   spanId: s6  parentSpanId: s2                           │
│  │   │   └── Span 7: ML Model inference            380ms  ✔     │
│  │   │                                                          │
│  │   └── Span 8: Shipping Est.  GET /estimate      7601ms ✗     │
│  │       spanId: s8  parentSpanId: s2                           │
│  │       └── Span 9: Maps API  GET /directions     7580ms ✗     │
│  │           status: SLOW (no timeout configured!)              │
│  │                                                              │
│  └── Span 10: Response sent to browser             2ms          │
│                                                                 │
│  → Bottleneck identificado instantáneamente: Span 9             │
│    Maps API externa sin timeout → bloquea todo el checkout      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Cómo funciona: Context Propagation paso a paso

```
┌─────────────────────────────────────────────────────────────────┐
│     CONTEXT PROPAGATION — W3C TRACE CONTEXT (HTTP)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. API Gateway recibe request del browser:                     │
│     → Genera: traceId=a3f7-bc12  spanId=s1                      │
│     → Crea root span (s1): "POST /checkout"                     │
│     → Llama a Orders con header:                                │
│       traceparent: 00-a3f7bc129d44...-s1-01                     │
│       (formato: version-traceId-parentSpanId-flags)             │
│                                                                 │
│  2. Orders Service recibe la llamada:                           │
│     → Lee el header traceparent                                 │
│     → Extrae: traceId=a3f7-bc12, parentSpanId=s1                │
│     → Crea span hijo (s2): "POST /orders", parent=s1            │
│     → Al llamar a Inventory, propaga header con s2 como parent: │
│       traceparent: 00-a3f7bc129d44...-s2-01                     │
│                                                                 │
│  3. Cada servicio en la cadena:                                 │
│     → Lee el header → crea span hijo → propaga header           │
│     → Todos los spans tienen el mismo traceId                   │
│     → El árbol se forma por la cadena parentSpanId              │
│                                                                 │
│  4. Cada span se reporta al Collector (OpenTelemetry):          │
│     → Al finalizar el span, el SDK lo envía al OTel Collector   │
│     → El Collector los agrega y los envía a Jaeger/Zipkin       │
│     → Jaeger reconstruye el árbol completo                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Stack tecnológico

```
┌─────────────────────────────────────────────────────────────────┐
│              STACK DE DISTRIBUTED TRACING — SHOPFAST            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Orders] [Payments] [Inventory] [Fraud] [Shipping] [Catalog]   │
│      │ OTel SDK   │ OTel SDK        │ OTel SDK       │ OTel SDK  │
│      └────────────┴─────────────────┴────────────────┘         │
│                           │                                     │
│              [OpenTelemetry Collector]                           │
│              Recibe spans (gRPC/HTTP)                            │
│              Aplica sampling rules                               │
│              Exporta a backend(s)                                │
│                           │                                     │
│              ┌────────────┴────────────┐                        │
│          [Jaeger]                  [Zipkin]                      │
│          Backend de trazas          Alternativa más simple       │
│          UI: flame graphs,          Elasticsearch / Cassandra    │
│          service maps,              como storage                 │
│          búsqueda por traceId,                                   │
│          servicio, duración,        Alternativas cloud:          │
│          status de error            → AWS X-Ray                  │
│                                     → Datadog APM               │
│                                     → New Relic                  │
│                                     → Tempo (Grafana)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Beneficios y desventajas — Distributed Tracing

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Visibilidad end-to-end:      │ Overhead de instrumentación:     │
│ el recorrido completo de     │ crear, propagar y reportar spans │
│ una request en todos los     │ añade ~1-5ms y CPU adicional     │
│ servicios, en una sola vista.│ por request.                     │
│                              │                                  │
│ Identificación de            │ Requiere instrumentar todos los  │
│ bottlenecks en segundos:     │ servicios — un servicio no       │
│ el span más largo es el      │ instrumentado crea un hueco      │
│ problema, sin búsqueda       │ ciego en la traza.               │
│ manual en logs.              │                                  │
│                              │                                  │
│ Service dependency map:      │ Sampling pierde trazas: con      │
│ Jaeger genera                │ head-based 1%, bugs esporádicos  │
│ automáticamente el grafo     │ pueden no capturarse nunca.      │
│ de dependencias entre        │ Tail-based es mejor pero más     │
│ servicios a partir de las    │ costoso de operar.               │
│ trazas reales.               │                                  │
│                              │                                  │
│ Correlación con logs:        │ Alto costo de storage: trazas    │
│ el traceId en los logs       │ completas con muchos spans son   │
│ permite ir directamente      │ voluminosas — necesitan          │
│ del span con error a los     │ Elasticsearch/Cassandra como     │
│ logs exactos de ese span.    │ backend + retención limitada.    │
│                              │                                  │
│ Performance profiling:       │ Complejidad operacional del      │
│ análisis de P99 por          │ stack: OTel Collector + Jaeger   │
│ operación individual,        │ + storage backend es otro        │
│ no solo por servicio.        │ sistema a mantener.              │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 3. Challenges in Event-Driven Architecture

**Español:**
El Distributed Tracing fue diseñado originalmente para sistemas de comunicación síncrona (HTTP/gRPC), donde la propagación del contexto es natural: cada llamada HTTP puede llevar el `traceparent` header. En una **arquitectura event-driven**, la comunicación es asíncrona y desacoplada — el producer publica un evento en Kafka y no sabe quién ni cuándo lo va a consumir. Esto rompe el modelo clásico de propagación de contexto y genera desafíos únicos para mantener la trazabilidad.

**English:**
Distributed Tracing was originally designed for synchronous communication systems (HTTP/gRPC), where context propagation is natural: every HTTP call can carry the `traceparent` header. In an **event-driven architecture**, communication is asynchronous and decoupled — the producer publishes an event to Kafka and does not know who will consume it or when. This breaks the classic context propagation model and generates unique challenges for maintaining traceability.

---

### Desafío 1 — Context Propagation a través de mensajes asíncronos

**Español:**
Cuando Orders Service publica un evento `order.created` en Kafka, la traza de la request HTTP que generó ese evento termina. El Notifications Service que consume ese evento horas o segundos después inicia una nueva traza sin conexión con la original. Sin instrumentación específica, las trazas de Orders y Notifications son dos árboles separados — no se puede ver que la notificación fue consecuencia de esa orden específica.

**English:**
When Orders Service publishes an `order.created` event to Kafka, the trace of the HTTP request that generated that event ends. The Notifications Service that consumes that event seconds or hours later starts a new trace with no connection to the original. Without specific instrumentation, the traces of Orders and Notifications are two separate trees — you cannot see that the notification was a consequence of that specific order.

```
┌─────────────────────────────────────────────────────────────────┐
│     CONTEXT PROPAGATION EN KAFKA — SHOPFAST                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SIN propagación de contexto:                                   │
│                                                                 │
│  Trace A: a3f7-bc12                                             │
│  API Gateway → Orders → [publica event] ──▶ Kafka               │
│                                                                 │
│  (tiempo después)                                               │
│                                                                 │
│  Trace B: 9f21-dd45  ← nueva traza, sin relación visible        │
│  Notifications ← [consume event] ──────── Kafka                 │
│                                                                 │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  CON propagación de contexto en Kafka message headers:          │
│                                                                 │
│  Orders al publicar el evento incluye en los Kafka headers:     │
│  traceparent: 00-a3f7bc12...-s2-01   ← mismo traceId            │
│                                                                 │
│  Notifications al consumir el evento:                           │
│  → Lee los Kafka headers                                        │
│  → Extrae traceId=a3f7-bc12, parentSpanId=s2                    │
│  → Crea un span HIJO dentro de la misma traza                   │
│  → La traza ahora abarca tanto la request HTTP como             │
│    el procesamiento asíncrono del evento                        │
│                                                                 │
│  Trace A: a3f7-bc12  (completo, end-to-end)                     │
│  API Gateway → Orders → [event] → Notifications → Email sent    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Desafío 2 — Fan-out: un evento, múltiples consumers

**Español:**
En un sistema event-driven, un solo evento puede ser consumido por múltiples servicios en paralelo. Cuando Orders publica `order.created`, pueden consumirlo simultáneamente: Notifications, Inventory, Fraud Detection, Analytics y Loyalty. Con context propagation, todos estos consumers crearían spans hijos dentro de la misma traza. Esto produce una traza con **múltiples ramas paralelas** que puede ser muy difícil de visualizar y puede generar miles de spans para un solo evento de alta importancia.

**English:**
In an event-driven system, a single event can be consumed by multiple services in parallel. When Orders publishes `order.created`, it may be consumed simultaneously by: Notifications, Inventory, Fraud Detection, Analytics, and Loyalty. With context propagation, all these consumers would create child spans within the same trace. This produces a trace with **multiple parallel branches** that can be very difficult to visualize and can generate thousands of spans for a single high-importance event.

```
┌─────────────────────────────────────────────────────────────────┐
│      FAN-OUT — order.created CON 5 CONSUMERS                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Trace: a3f7-bc12                                               │
│                                                                 │
│  API Gateway → Orders → publica order.created                   │
│                              │                                  │
│                              ├──▶ Notifications (email)         │
│                              │    └── Span: send_email  180ms   │
│                              │                                  │
│                              ├──▶ Inventory                     │
│                              │    └── Span: reserve_stock  95ms │
│                              │                                  │
│                              ├──▶ Fraud Detection               │
│                              │    └── Span: ml_check  420ms     │
│                              │                                  │
│                              ├──▶ Analytics                     │
│                              │    └── Span: record_event  12ms  │
│                              │                                  │
│                              └──▶ Loyalty                       │
│                                   └── Span: add_points  30ms    │
│                                                                 │
│  Una sola traza con branches paralelas de 5 servicios.          │
│  Visualizar el flame graph puede ser confuso para el equipo.    │
│                                                                 │
│  Estrategia alternativa: crear una NUEVA traza por consumer     │
│  y usar un "link" (no parent) para referenciar la traza origen. │
│  Soportado por OpenTelemetry con SpanLinks.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Desafío 3 — Sampling en flujos asíncronos

**Español:**
En flujos síncronos, la decisión de sampling se toma una vez en el API Gateway y se propaga. En flujos asíncronos con Kafka, el producer puede haber decidido NO guardar la traza (sampling out), pero cuando el consumer procesa el mensaje, ¿debe crearse un nuevo span o no? Si el consumer ignora la decisión de sampling del producer y crea siempre trazas, se rompe la coherencia del sampling. Si el consumer respeta la decisión de sampling del producer, eventos erróneos que ocurren solo en el consumer pueden no capturarse nunca.

**English:**
In synchronous flows, the sampling decision is made once at the API Gateway and propagated. In asynchronous flows with Kafka, the producer may have decided NOT to save the trace (sampling out), but when the consumer processes the message, should a new span be created or not? If the consumer ignores the producer's sampling decision and always creates traces, sampling coherence is broken. If the consumer respects the producer's sampling decision, error events that occur only in the consumer may never be captured.

```
┌─────────────────────────────────────────────────────────────────┐
│      SAMPLING EN FLUJOS ASÍNCRONOS — DILEMA                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Escenario:                                                     │
│  → Orders procesa 1000 órdenes/minuto                           │
│  → Head-based sampling: 1% → solo 10 trazas/minuto guardadas   │
│  → El evento de la orden #421 fue sampled OUT (no guardado)     │
│  → Notifications falla al procesar el evento de la orden #421  │
│                                                                 │
│  Problema:                                                      │
│  → El error de Notifications no tiene traza padre              │
│  → Si Notifications crea una nueva traza: no hay contexto       │
│    de qué orden fue, qué usuario, qué items                    │
│  → El incidente es difícil de reproducir y diagnosticar        │
│                                                                 │
│  Soluciones:                                                    │
│  1. Tail-based sampling: el collector guarda el 100% de         │
│     trazas con errores, independientemente del sampling del     │
│     producer. Más costoso pero más confiable.                   │
│                                                                 │
│  2. Incluir en el Kafka header el traceId aunque la traza       │
│     sea sampled-out (flag "sampled=0"), para que el consumer    │
│     pueda decidir independientemente si registrar el error.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Distributed Tracing Solutions

### Distributed Tracing Instrumentation Frameworks

**OpenTelemetry** - OpenTelemetry is a is a collection of APIs, SDKs, and tools for instrumenting, collecting, and exporting metrics, logs, and traces. It is open-source and is available in many programming languages such as C++, #/.NET, Erlang/Elixir, Go, Java, JavaScript, PHP, Python, Ruby, Rust, and Swift.

OpenTelemetry is also a specification that describes the cross-language requirements and expectations for all OpenTelemetry implementations.

### Distributed Tracing Backends

**Jaeger** - Open source, distributed tracing platform that is cloud-native, infinitely scalable, and 100% free. It enables the monitoring of distributed workflows, tracking down root causes for performance bottlenecks, and analyzing service dependencies.

Requies OpenTelemetry for instrumentation .

**Zipkin** - Another open-source distributed tracing system. Its data served to the UI is stored in memory or persistently within Apache Cassandra or Elasticsearch. Originally developed at Twitter in 2010 and based on Google's Dapper papers.

Supports a variety of official and community-created instrumentation frameworks.

**Uptrace** - Uptrace is an OpenTelemetry-based observability platform that helps you monitor, understand, and optimize distributed systems.

It is a commercial, cloud-based solution that supports a variety of features such as App Performance Monitoring, metrics collection and visualization, logs injection and analysis, and more.

---

### Desafío 4 — Trazas de larga duración (long-running traces)

**Español:**
En sistemas event-driven, algunas trazas pueden durar horas o días. Un proceso de fulfillment de ShopFast puede ser: `order.created` → (procesamiento inmediato) → `order.shipped` (2 horas después) → `order.delivered` (3 días después). Vincular todos estos eventos en una sola traza genera una traza de duración extremadamente larga, con spans separados por horas. Los backends de tracing como Jaeger no están diseñados para trazas de estas duraciones — pueden expirar o ser difíciles de visualizar.

**English:**
In event-driven systems, some traces can last hours or days. A ShopFast fulfillment process can be: `order.created` → (immediate processing) → `order.shipped` (2 hours later) → `order.delivered` (3 days later). Linking all these events in a single trace generates an extremely long-duration trace, with spans separated by hours. Tracing backends like Jaeger are not designed for traces of these durations — they can expire or be difficult to visualize.

```
┌─────────────────────────────────────────────────────────────────┐
│     LONG-RUNNING TRACES — ESTRATEGIAS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Opción A: Una sola traza larga                                 │
│  → Traza dura 3 días                                            │
│  → Jaeger timeout: 24 horas → la traza se pierde               │
│  → El flame graph es ilegible (spans separados por días)        │
│  ✗ No recomendado para eventos de negocio de larga duración     │
│                                                                 │
│  Opción B: Trazas separadas + correlación por businessId        │
│  → Cada etapa del fulfillment tiene su propia traza             │
│  → Todas las trazas incluyen orderId como atributo              │
│  → Para correlacionar: buscar en Jaeger por orderId="421"       │
│  → Cada traza es corta y manejable                              │
│  ✔ Recomendado para workflows de negocio de larga duración      │
│                                                                 │
│  Opción C: SpanLinks (OpenTelemetry)                            │
│  → Cada traza es independiente                                  │
│  → Un span puede tener un "link" a otro span de otra traza      │
│  → Mantiene la correlación sin extender la duración de la traza │
│  ✔ Solución nativa de OTel para este problema                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Resumen de desafíos y soluciones

| Desafío | Causa raíz | Solución recomendada |
|---------|-----------|---------------------|
| Context propagation asíncrono | Kafka no propaga headers automáticamente | OpenTelemetry SDK con instrumentación Kafka: propaga `traceparent` en Kafka message headers |
| Fan-out con múltiples consumers | Una traza se bifurca en N branches paralelas | Usar **SpanLinks** en vez de spans hijos para los consumers asíncronos |
| Sampling incoherente en flujos asíncronos | Producer sampled-out, consumer tiene error | **Tail-based sampling** que guarda 100% de trazas con errores |
| Long-running traces | Workflows de negocio que duran horas o días | Trazas separadas por etapa + correlación por `businessId` como atributo |

> **Regla de Pogrebinsky:** "Distributed tracing in event-driven systems is fundamentally different from synchronous tracing. The asynchronous, decoupled nature of messaging breaks the assumptions of classical trace propagation. You must explicitly decide: do I want a single trace that spans the entire async workflow, or separate traces correlated by a business ID? There is no universally right answer — it depends on the duration and complexity of the workflow."