# Message Delivery Semantics in Event-Driven Architecture

---

## 1. El problema de la entrega de mensajes

**Español:**
En un sistema distribuido, enviar un mensaje de un servicio a otro parece una operación simple. En la práctica, entre el producer y el consumer existen múltiples puntos de fallo independientes: la red puede fallar, el broker puede reiniciarse, el consumer puede caerse en el peor momento posible. La pregunta fundamental de las **delivery semantics** es: ¿qué garantías ofrece el sistema sobre cuántas veces llegará un mensaje a su destino?

**English:**
In a distributed system, sending a message from one service to another seems like a simple operation. In practice, between the producer and the consumer there are multiple independent failure points: the network can fail, the broker can restart, the consumer can crash at the worst possible moment. The fundamental question of **delivery semantics** is: what guarantees does the system offer about how many times a message will reach its destination?

### Por qué es más difícil de lo que parece

```
┌─────────────────────────────────────────────────────────────────┐
│         PUNTOS DE FALLO EN LA ENTREGA DE UN MENSAJE             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer                 Broker                  Consumer      │
│  ────────                ────────                ──────────     │
│     │                      │                        │           │
│     │──── send ────────▶   │                        │           │
│     │         ❌ red cae   │                        │           │
│     │         (¿llegó o no llegó?)                  │           │
│     │                      │                        │           │
│     │──── send ────────▶   │ ❌ broker cae antes    │           │
│     │                      │    de persistir        │           │
│     │                      │                        │           │
│     │──── send ────────▶   │──── deliver ───────▶   │           │
│     │                      │              ❌ consumer cae       │
│     │                      │               antes de procesar    │
│     │                      │                        │           │
│     │──── send ────────▶   │──── deliver ───────▶   │           │
│     │                      │                        │ procesa   │
│     │                      │              ❌ consumer cae       │
│     │                      │               DESPUÉS de procesar  │
│     │                      │               pero ANTES del ACK   │
│     │                      │──── re-deliver ──────▶ │           │
│     │                      │                    DUPLICADO ⚠️    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Español:**
El escenario más problemático es el último: el consumer procesó el mensaje correctamente, pero falló antes de confirmar al broker. El broker no puede saber si el consumer procesó o no — solo sabe que no recibió el ACK. Su única opción segura es **reenviar el mensaje**. Este dilema es la raíz de todos los problemas de delivery semantics en sistemas distribuidos.

**English:**
The most problematic scenario is the last one: the consumer correctly processed the message but crashed before acknowledging the broker. The broker cannot know whether the consumer processed it or not — it only knows it did not receive the ACK. Its only safe option is to **resend the message**. This dilemma is the root of all delivery semantics problems in distributed systems.

> **Regla de Pogrebinsky:** "In distributed systems, you cannot have both 'no message loss' and 'no duplicates' without paying a significant cost in complexity and performance. You must choose your trade-off deliberately."

---

## 2. Problemas de entrega de eventos en EDA

**Español:**
En una arquitectura orientada a eventos, los problemas de entrega se manifiestan en cuatro escenarios concretos. Cada uno ocurre en un punto distinto del camino Producer → Broker → Consumer, y cada uno tiene consecuencias distintas para el negocio.

**English:**
In an event-driven architecture, delivery problems manifest in four concrete scenarios. Each occurs at a different point along the Producer → Broker → Consumer path, and each has different consequences for the business.

### Problema 1 — Pérdida antes del broker (producer-side failure)

```
  ShopFast — Orders Service publica order.created

  Orders Service
       │
       │  crash aquí ❌
       │  (justo antes de llamar a broker.publish())
       │
    [orden guardada en DB]
    [evento NUNCA publicado]
       │
       ▼
    Broker no recibe nada.
    Payments, Inventory, Shipping → nunca se enteran de la orden.
    El cliente recibió confirmación → la orden no se procesa.
```

**Español:**
Este es el problema del **dual write**: escribir en la base de datos y publicar el evento son dos operaciones separadas. Si el proceso muere entre las dos, el sistema queda en un estado inconsistente. La solución a este problema específico es el **Outbox Pattern** (patrón de bandeja de salida), que garantiza que si la DB se escribió, el evento se publicará eventualmente.

**English:**
This is the **dual write** problem: writing to the database and publishing the event are two separate operations. If the process dies between them, the system is left in an inconsistent state. The solution to this specific problem is the **Outbox Pattern**, which guarantees that if the DB was written, the event will eventually be published.

### Problema 2 — Pérdida en tránsito (network failure)

```
  Orders ──── publish(order.created) ────▶ [RED] ❌ ──▶ Broker

  El broker nunca recibe el mensaje.
  Orders cree que lo envió (no hubo excepción de red).
  El evento se pierde silenciosamente.

  Solución: producer espera ACK del broker antes de continuar.
  Si no recibe ACK → reintenta.
  Riesgo de reintento: si el broker SÍ recibió pero el ACK se perdió
  → el producer reenvía → DUPLICADO en el broker.
```

### Problema 3 — Fallo del consumer antes de procesar

```
  Broker ──── deliver(order.created) ────▶ Payments Service
                                                │
                                           ❌ crash aquí
                                           (antes de procesar)

  El mensaje llega a Payments pero no se procesa.
  Broker no recibió ACK → considera el mensaje no entregado.
  Broker reintenta la entrega (at-least-once).
  → Payments procesa en el siguiente reintento. ✔

  Consecuencia: retraso en el procesamiento, pero sin pérdida.
```

### Problema 4 — Fallo del consumer después de procesar (el más peligroso)

```
  Broker ──── deliver(order.created) ────▶ Payments Service
                                                │
                                           procesa el cobro ✔
                                           (tarjeta cargada)
                                                │
                                           ❌ crash aquí
                                           (antes de enviar ACK)

  Broker no recibió ACK → reenvía el mensaje.
  Payments vuelve online → recibe order.created OTRA VEZ.
  Payments cobra la tarjeta por SEGUNDA VEZ. ⚠️

  Este es el escenario que exige IDEMPOTENCIA en el consumer.
```

### Resumen de escenarios de fallo

| Escenario | Punto de fallo | Consecuencia | Solución |
|-----------|---------------|--------------|----------|
| Producer cae antes de publicar | Producer → Broker | Pérdida silenciosa | Outbox Pattern |
| Red falla al publicar | Producer → Broker | Pérdida o duplicado | Producer ACK + retry |
| Consumer cae antes de procesar | Broker → Consumer | Retraso, sin pérdida | At-least-once |
| Consumer cae después de procesar | Consumer → Broker ACK | **Duplicado** ⚠️ | Idempotencia obligatoria |

---

## 3. Semánticas At-Most-Once y At-Least-Once

### At-Most-Once — "como máximo una vez"

**Español:**
El producer envía el mensaje **sin esperar confirmación** del broker. Si la red falla o el broker cae, el mensaje se pierde sin reintentos. El consumer procesa el mensaje y **descarta el ACK** — el broker no sabe si fue procesado. Garantiza **cero duplicados** a cambio de permitir pérdidas.

**English:**
The producer sends the message **without waiting for a confirmation** from the broker. If the network fails or the broker goes down, the message is lost with no retries. The consumer processes the message and **discards the ACK** — the broker does not know whether it was processed. It guarantees **zero duplicates** at the cost of allowing losses.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AT-MOST-ONCE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRODUCER SIDE:                                                 │
│    producer.send(event)  ← no espera ACK del broker            │
│    // continúa de inmediato                                     │
│    // si el broker no recibió → mensaje perdido, sin reintento  │
│                                                                 │
│  En Kafka:  acks = 0                                            │
│    → el producer no espera ninguna confirmación del broker      │
│    → máximo throughput, mínima latencia                         │
│    → pérdida posible si el líder de la partición cae            │
│                                                                 │
│  CONSUMER SIDE:                                                 │
│    1. consumer recibe el mensaje                                │
│    2. consumer envía ACK al broker (auto-commit inmediato)      │
│    3. consumer procesa el mensaje                               │
│    → si el consumer cae entre 2 y 3: mensaje perdido            │
│                                                                 │
│  Garantía: 0 o 1 entrega                                        │
│  Pérdida:  Posible                                              │
│  Duplicado: Imposible                                           │
│  Throughput: Máximo                                             │
└─────────────────────────────────────────────────────────────────┘
```

**Español:**
El orden incorrecto de operaciones en el consumer es la clave: el ACK se envía **antes** de procesar. Esto garantiza que nunca habrá duplicados — pero si el consumer cae después del ACK y antes de procesar, el mensaje se pierde definitivamente.

**English:**
The incorrect order of operations on the consumer side is the key: the ACK is sent **before** processing. This guarantees there will never be duplicates — but if the consumer crashes after the ACK and before processing, the message is lost for good.

```
  Casos de uso en ShopFast — cuándo At-Most-Once es aceptable:

  ✔ user.product.viewed       → analytics de comportamiento
  ✔ page.load.time            → métricas de performance
  ✔ search.query.logged       → log de búsquedas para ML
  ✔ recommendation.shown      → impresiones de recomendaciones

  En todos estos casos: perder 1 evento de cada 100,000
  no impacta el negocio. La agregación absorbe las pérdidas.
```

---

### At-Least-Once — "al menos una vez"

**Español:**
El producer espera el **ACK del broker** antes de considerar el mensaje enviado. Si no recibe ACK, **reintenta**. El consumer procesa el mensaje **antes** de enviar el ACK. Si el consumer cae después de procesar y antes del ACK, el broker reenvía y el consumer procesa dos veces. Garantiza **cero pérdidas** a cambio de permitir duplicados. **El consumer debe ser idempotente.**

**English:**
The producer waits for the **broker ACK** before considering the message sent. If no ACK is received, it **retries**. The consumer processes the message **before** sending the ACK. If the consumer crashes after processing and before the ACK, the broker resends and the consumer processes twice. It guarantees **zero loss** at the cost of allowing duplicates. **The consumer must be idempotent.**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AT-LEAST-ONCE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRODUCER SIDE:                                                 │
│    producer.send(event)                                         │
│    espera ACK del broker...                                     │
│    si timeout o error → REINTENTA                               │
│                                                                 │
│  En Kafka:  acks = all  (todos los replicas confirman)          │
│             retries = Integer.MAX_VALUE                         │
│             delivery.timeout.ms = 120000                        │
│                                                                 │
│  CONSUMER SIDE:                                                 │
│    1. consumer recibe el mensaje                                │
│    2. consumer PROCESA el mensaje ← primero procesa             │
│    3. consumer envía ACK al broker ← luego confirma             │
│    → si cae entre 2 y 3: broker reenvía → DUPLICADO ⚠️         │
│    → el consumer debe manejar el duplicado con idempotencia     │
│                                                                 │
│  En Kafka: enable.auto.commit = false                           │
│            consumer.commitSync() solo después de procesar       │
│                                                                 │
│  Garantía: ≥1 entrega                                           │
│  Pérdida:  Imposible                                            │
│  Duplicado: Posible → IDEMPOTENCIA OBLIGATORIA                  │
│  Throughput: Alto                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Técnicas de idempotencia para at-least-once

**Español:**
La idempotencia significa que procesar el mismo mensaje N veces produce el mismo resultado que procesarlo una sola vez. Hay tres técnicas principales para implementarla en los consumers.

**English:**
Idempotency means that processing the same message N times produces the same result as processing it once. There are three main techniques to implement it in consumers.

```
  TÉCNICA 1 — Unique Constraint en la base de datos
  ──────────────────────────────────────────────────────────────
  Payments Service recibe order.created { orderId: "421" }

  CREATE TABLE payments (
    order_id VARCHAR PRIMARY KEY,  ← constraint único
    charge_id VARCHAR,
    processed_at TIMESTAMP
  );

  1ª entrega: INSERT INTO payments(order_id='421', ...) → OK ✔
  2ª entrega: INSERT INTO payments(order_id='421', ...)
              → UNIQUE VIOLATION → catch → ignorar → OK ✔

  El resultado es el mismo: exactamente 1 cobro para la orden 421.


  TÉCNICA 2 — Idempotency Key explícita
  ──────────────────────────────────────────────────────────────
  El evento incluye un campo eventId único:
  { eventId: "evt-9f3a2c1d", orderId: "421", ... }

  Consumer mantiene tabla de eventos procesados:
  CREATE TABLE processed_events (
    event_id VARCHAR PRIMARY KEY,
    processed_at TIMESTAMP
  );

  Al recibir un evento:
  1. SELECT COUNT(*) FROM processed_events WHERE event_id = ?
  2. Si existe → ya procesado → ignorar
  3. Si no existe → procesar + INSERT INTO processed_events


  TÉCNICA 3 — Operaciones naturalmente idempotentes
  ──────────────────────────────────────────────────────────────
  En lugar de:  balance = balance + 10  (no idempotente)
  Usar:         balance = 150           (SET absoluto, idempotente)

  En lugar de:  INSERT INTO cart_items VALUES (...)
  Usar:         INSERT ... ON CONFLICT (user_id, product_id)
                DO UPDATE SET quantity = excluded.quantity
```

### Comparación At-Most-Once vs At-Least-Once

```
┌───────────────────────┬───────────────────────┬───────────────────────┐
│ Dimensión             │ At-Most-Once          │ At-Least-Once         │
├───────────────────────┼───────────────────────┼───────────────────────┤
│ Garantía              │ 0 o 1 entrega         │ ≥1 entrega            │
│ Pérdida de mensajes   │ Posible               │ Imposible             │
│ Duplicados            │ Imposible             │ Posible               │
│ Requiere idempotencia │ No                    │ Sí — obligatorio      │
│ Complejidad consumer  │ Mínima                │ Media                 │
│ Throughput producer   │ Máximo (acks=0)       │ Alto (acks=all)       │
│ Latencia              │ Mínima                │ Ligeramente mayor     │
│ Cuándo usar           │ Eventos no críticos   │ Eventos de negocio    │
│ Ejemplo ShopFast      │ page views, métricas  │ order.created, pago   │
└───────────────────────┴───────────────────────┴───────────────────────┘
```

---

## 4. Semántica Exactly-Once

**Español:**
Exactly-once es la semántica más deseada y la más difícil de conseguir: cada mensaje se entrega y procesa **exactamente una vez** — sin pérdidas ni duplicados. El problema fundamental es que garantizar esto requiere **coordinación atómica** entre el producer, el broker y el consumer, tres sistemas independientes que pueden fallar de forma independiente.

**English:**
Exactly-once is the most desired and hardest to achieve semantic: each message is delivered and processed **exactly once** — with no losses or duplicates. The fundamental problem is that guaranteeing this requires **atomic coordination** among the producer, broker, and consumer, three independent systems that can fail independently.

### Por qué es tan difícil

```
┌─────────────────────────────────────────────────────────────────┐
│              EL DILEMA DEL EXACTLY-ONCE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Para garantizar exactly-once se necesita:                      │
│                                                                 │
│  1. El producer no puede publicar duplicados al broker          │
│     (si reintenta tras un timeout, ¿el broker ya lo tiene?)     │
│                                                                 │
│  2. El broker no puede entregar el mismo mensaje dos veces      │
│     al mismo consumer group                                     │
│                                                                 │
│  3. El consumer no puede procesar el mismo mensaje dos veces    │
│     aunque el broker lo entregue dos veces                      │
│                                                                 │
│  4. El procesamiento del consumer y el ACK al broker deben      │
│     ser ATÓMICOS — o ambos ocurren, o ninguno ocurre            │
│                                                                 │
│  El problema: el procesamiento (ej: INSERT en PostgreSQL) y     │
│  el ACK (commit de offset en Kafka) son dos sistemas distintos. │
│  No existe un gestor de transacciones distribuidas que los       │
│  coordine de forma nativa.                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Kafka Exactly-Once: Idempotent Producer + Transactions

**Español:**
Kafka implementa exactly-once en dos capas: **idempotent producer** para eliminar duplicados en la escritura al broker, y **transactional API** para hacer que múltiples operaciones sean atómicas.

**English:**
Kafka implements exactly-once in two layers: **idempotent producer** to eliminate duplicates when writing to the broker, and **transactional API** to make multiple operations atomic.

```
  CAPA 1 — Idempotent Producer
  ──────────────────────────────────────────────────────────────
  Configuración:  enable.idempotence = true

  Kafka asigna a cada producer un PID (Producer ID).
  Cada mensaje incluye un sequence number (PID + sequence).

  Producer envía:  { PID: 42, seq: 100, payload: order.created }
  Red falla → producer reintenta
  Producer reenvía: { PID: 42, seq: 100, payload: order.created }

  Broker detecta: PID=42, seq=100 ya fue recibido → DESCARTA duplicado.
  El producer puede reintentar de forma segura.


  CAPA 2 — Transactional Producer
  ──────────────────────────────────────────────────────────────
  Configuración:  transactional.id = "orders-producer-1"

  producer.beginTransaction()
    producer.send("orders-topic",     order.created)
    producer.send("inventory-topic",  inventory.reserve)
    producer.send("payments-topic",   payment.charge)
  producer.commitTransaction()

  → Las 3 publicaciones son ATÓMICAS:
    o se publican las 3, o ninguna (si commitTransaction falla).
  → Los consumers con isolation.level = read_committed
    solo ven mensajes de transacciones completadas.
```

### El trade-off del Exactly-Once en Kafka

```
┌─────────────────────────────────────────────────────────────────┐
│              EXACTLY-ONCE — COSTOS Y BENEFICIOS                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Beneficios:                                                    │
│  ✔ Garantía absoluta: 0 pérdidas + 0 duplicados                 │
│  ✔ No requiere idempotencia en el consumer                      │
│  ✔ Múltiples topics se actualizan atómicamente                  │
│                                                                 │
│  Costos:                                                        │
│  ✗ Latencia mayor: el broker coordina con todos los replicas    │
│    antes de marcar la transacción como committed                │
│  ✗ Throughput reducido: ~20-30% menos que at-least-once         │
│  ✗ Configuración compleja: transactional.id único por producer  │
│    instance, gestión de zombie producers                        │
│  ✗ Consumer debe usar isolation.level = read_committed          │
│    (por defecto es read_uncommitted → no garantiza EO)          │
│  ✗ Solo funciona dentro del ecosistema Kafka                    │
│    (si el consumer escribe en PostgreSQL, no hay garantía EO    │
│     entre Kafka y PostgreSQL)                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### La alternativa práctica: At-Least-Once + Idempotencia

**Español:**
En la práctica, la mayoría de los sistemas de producción de alta disponibilidad **no usan exactly-once de Kafka** para su flujo principal. En cambio, usan at-least-once con consumers idempotentes, lo que logra el mismo resultado observable (ningún efecto duplicado) con menor complejidad operacional y mayor throughput.

**English:**
In practice, most high-availability production systems **do not use Kafka's exactly-once** for their main flow. Instead, they use at-least-once with idempotent consumers, which achieves the same observable result (no duplicate effects) with lower operational complexity and higher throughput.

```
  EXACTLY-ONCE real de Kafka:
  ────────────────────────────────────────────────────────────
  Garantía:    exactamente 1 entrega Y exactamente 1 proceso
  Mecanismo:   Kafka transactions + idempotent producer
  Throughput:  -20% respecto a at-least-once
  Complejidad: Alta

  AT-LEAST-ONCE + IDEMPOTENT CONSUMER (alternativa práctica):
  ────────────────────────────────────────────────────────────
  Garantía:    ≥1 entrega PERO resultado = exactamente 1 efecto
  Mecanismo:   unique constraint / idempotency key en el consumer
  Throughput:  máximo
  Complejidad: Media (lógica de deduplicación en el consumer)

  Resultado observable para el negocio: IDÉNTICO.
  La tarjeta se cobra exactamente una vez en ambos casos.
```

### Cuándo usar cada semántica — guía de decisión ShopFast

```
┌─────────────────────────────────────────────────────────────────┐
│  Evento                    │ Semántica recomendada              │
├────────────────────────────┼────────────────────────────────────┤
│  page.viewed               │ At-most-once                       │
│  search.performed          │ At-most-once                       │
│  user.product.viewed       │ At-most-once                       │
│  recommendation.shown      │ At-most-once                       │
├────────────────────────────┼────────────────────────────────────┤
│  order.created             │ At-least-once + idempotencia       │
│  inventory.reserved        │ At-least-once + idempotencia       │
│  shipping.created          │ At-least-once + idempotencia       │
│  user.registered           │ At-least-once + idempotencia       │
│  notification.sent         │ At-least-once + idempotencia       │
├────────────────────────────┼────────────────────────────────────┤
│  payment.processed         │ At-least-once + idempotencia       │
│  (alternativa: EO Kafka)   │ (o Exactly-once si regulación      │
│                            │  financiera lo exige explícitamente)│
│  account.balance.updated   │ Exactly-once (Kafka transactions)  │
│  loyalty.points.credited   │ At-least-once + idempotencia       │
└────────────────────────────┴────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Exactly-once sounds like the obvious choice — but its cost in throughput and complexity is real. In most cases, at-least-once with idempotent consumers gives you the same business guarantee at a fraction of the cost. Reserve exactly-once for the cases where the regulatory or financial risk of a duplicate effect is genuinely unacceptable."

---

## 5. Garantías de entrega por tecnología de broker

**Español:**
No todos los brokers implementan las delivery semantics de la misma forma. Conocer las garantías nativas de cada plataforma permite elegir la herramienta correcta para cada caso de uso y evitar asumir garantías que el broker no ofrece.

**English:**
Not all brokers implement delivery semantics the same way. Knowing the native guarantees of each platform allows you to choose the right tool for each use case and avoid assuming guarantees that the broker does not provide.

### Apache Kafka

**Español:**
Kafka es el broker más flexible en cuanto a semánticas: el nivel de garantía se configura explícitamente tanto en el producer como en el consumer. Soporta las tres semánticas y es el único broker mainstream que ofrece exactly-once nativo dentro de su propio ecosistema.

**English:**
Kafka is the most flexible broker in terms of semantics: the guarantee level is configured explicitly on both the producer and consumer side. It supports all three semantics and is the only mainstream broker that offers native exactly-once within its own ecosystem.

```
┌─────────────────────────────────────────────────────────────────┐
│                    APACHE KAFKA                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer Delivery Semantics (configurable):                    │
│    acks = 0            → At-most-once                           │
│    acks = 1            → At-least-once (líder confirma)         │
│    acks = all          → At-least-once (todos los replicas)     │
│    enable.idempotence  → Idempotent producer (base de EO)       │
│    transactional.id    → Exactly-once (multi-topic atómico)     │
│                                                                 │
│  Consumer Receipt Semantics (configurable):                     │
│    enable.auto.commit = true   → At-most-once                   │
│    enable.auto.commit = false  → At-least-once                  │
│    + isolation.level = read_committed → Exactly-once            │
│                                                                 │
│  ⚠️ Nota importante:                                            │
│    Exactly-once en Kafka aplica a la transferencia de datos     │
│    ENTRE TOPICS de Kafka (Kafka Streams / Kafka Transactions).  │
│    Si el consumer escribe en PostgreSQL u otra DB externa,      │
│    la garantía EO no cubre esa operación externa.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Amazon SQS

**Español:**
SQS ofrece dos tipos de colas con garantías distintas. Las colas estándar garantizan at-least-once por diseño — los mensajes pueden llegar más de una vez. Las colas FIFO añaden exactly-once **solo al momento de publicar** (deduplicación en la ingesta), pero la semántica de procesamiento en el consumer sigue siendo at-least-once.

**English:**
SQS offers two queue types with different guarantees. Standard queues guarantee at-least-once by design — messages may arrive more than once. FIFO queues add exactly-once **only at publish time** (deduplication on ingestion), but the consumer processing semantic remains at-least-once.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AMAZON SQS                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SQS Standard Queues:                                           │
│    Delivery:     At-least-once                                  │
│    Ordering:     Best-effort (sin orden garantizado)            │
│    Throughput:   Casi ilimitado                                 │
│    Uso:          Casos donde el consumer ya es idempotente      │
│                                                                 │
│  SQS FIFO Queues:                                               │
│    Exactly-once processing — SOLO al publicar en la cola:       │
│    → SQS deduplica mensajes con el mismo MessageDeduplicationId │
│      dentro de una ventana de 5 minutos                         │
│    → Garantiza que el mismo mensaje no entra DOS VECES a la cola│
│    Ordering:     Estricto (FIFO garantizado por MessageGroupId) │
│    Throughput:   Hasta 3.000 msg/seg (con batching)             │
│                                                                 │
│  ⚠️ Nota: "Exactly-once processing" en FIFO se refiere a la    │
│    INGESTA en la cola, no al procesamiento del consumer.        │
│    El consumer sigue pudiendo recibir duplicados si falla       │
│    antes de eliminar el mensaje de la cola.                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Google Cloud Pub/Sub

**Español:**
Cloud Pub/Sub ofrece exactly-once delivery nativo, disponible como opción de configuración en la suscripción. Cuando está habilitado, el broker garantiza que el mismo mensaje no se entregará más de una vez al mismo subscriber, incluso ante reintentos internos. Esto simplifica el diseño de los consumers — no requieren lógica de deduplicación.

**English:**
Cloud Pub/Sub offers native exactly-once delivery, available as a configuration option on the subscription. When enabled, the broker guarantees that the same message will not be delivered more than once to the same subscriber, even with internal retries. This simplifies consumer design — they do not require deduplication logic.

```
┌─────────────────────────────────────────────────────────────────┐
│                 GOOGLE CLOUD PUB/SUB                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Exactly-once delivery (configurable por suscripción):          │
│    → Activar: enable_exactly_once_delivery = true               │
│    → El broker trackea qué mensajes ya fueron ACK-eados         │
│    → Re-entregas por retry interno no generan duplicados        │
│    → El consumer puede omitir lógica de deduplicación           │
│                                                                 │
│  Costo:                                                         │
│    → Mayor latencia en la entrega (coordinación interna)        │
│    → Precio por mensaje ligeramente mayor                       │
│                                                                 │
│  Caso de uso ShopFast:                                          │
│    → payment.processed: activar EO para evitar doble cobro      │
│    → user.product.viewed: dejar en at-least-once (costo menor)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Microsoft Azure — Event Hubs y Event Grid

**Español:**
Azure ofrece dos servicios de mensajería con modelos muy distintos. Event Hubs es un broker de streaming similar a Kafka — alto throughput, retención de eventos, múltiples consumers independientes. Event Grid es un bus de eventos serverless orientado a notificaciones, con retry automático pero sin garantía de orden estricto.

**English:**
Azure offers two messaging services with very different models. Event Hubs is a streaming broker similar to Kafka — high throughput, event retention, multiple independent consumers. Event Grid is a serverless event bus oriented toward notifications, with automatic retry but without strict ordering guarantees.

```
┌─────────────────────────────────────────────────────────────────┐
│                   MICROSOFT AZURE                               │
├──────────────────────────────┬──────────────────────────────────┤
│  Event Hubs                  │  Event Grid                      │
├──────────────────────────────┼──────────────────────────────────┤
│  Modelo: Streaming (= Kafka) │  Modelo: Push/Webhook            │
│  Entrega: Exactly-once       │  Entrega: At-least-once          │
│    (output binding)          │    (retry automático)            │
│  Retención: configurable     │  Retención: 24h (reintentos)     │
│  Consumers: múltiples,       │  Consumers: push a endpoints     │
│    independientes por offset │    HTTP, Azure Functions, etc.   │
│  Throughput: millones msg/s  │  Throughput: 10M eventos/seg     │
│  Orden: por partición        │  Orden: best-effort              │
├──────────────────────────────┼──────────────────────────────────┤
│  Uso en ShopFast:            │  Uso en ShopFast:                │
│  Streaming de user-behavior  │  Notificaciones de infraestructa │
│  Pipeline de Analytics/ML    │  order.created → trigger         │
│  Equivalente a Kafka topics  │  Azure Function de notificaciones│
└──────────────────────────────┴──────────────────────────────────┘
```

### Tabla comparativa entre brokers

| Broker | At-Most-Once | At-Least-Once | Exactly-Once | Notas |
|--------|-------------|--------------|-------------|-------|
| Apache Kafka | ✔ (acks=0) | ✔ (acks=all) | ✔ (transactions) | EO solo entre topics Kafka |
| Amazon SQS Standard | — | ✔ (nativo) | — | Consumer debe ser idempotente |
| Amazon SQS FIFO | — | ✔ | ✔ (solo ingesta) | EO en publicación, no en consume |
| Google Cloud Pub/Sub | — | ✔ (default) | ✔ (configurable) | EO activable por suscripción |
| Azure Event Hubs | — | ✔ | ✔ (output binding) | Similar a Kafka en garantías |
| Azure Event Grid | — | ✔ (retry) | — | Sin orden garantizado |

> **Regla de Pogrebinsky:** "The broker you choose constrains the delivery guarantees you can achieve. Kafka gives you maximum control at maximum operational cost. Managed services like SQS or Pub/Sub trade configurability for operational simplicity. Know what your broker guarantees natively — and what it delegates back to you."