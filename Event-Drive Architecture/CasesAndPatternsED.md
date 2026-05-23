# Use Cases and Patterns of Event-Driven Architecture

---

## 1. Casos de uso de la Arquitectura Orientada a Eventos / Use Cases of Event-Driven Architecture

**Español:**
EDA no es una solución universal — es la elección correcta en escenarios específicos donde el desacoplamiento, la resiliencia y la capacidad de reacción independiente de múltiples componentes son requisitos reales del negocio.

**English:**
EDA is not a universal solution — it is the right choice in specific scenarios where decoupling, resilience, and the ability for multiple components to react independently are real business requirements.

### Caso 1 — Flujos de trabajo asíncronos con múltiples consumidores

**Español:**
Cuando una acción de negocio desencadena reacciones en varios dominios distintos, EDA es la elección natural. El producer publica un único evento y cada dominio reacciona de forma autónoma.

**English:**
When a business action triggers reactions across several different domains, EDA is the natural choice. The producer publishes a single event and each domain reacts autonomously.

```
  ShopFast — order.created
       │
       ├──▶ Payments Service    → cobra la tarjeta
       ├──▶ Inventory Service   → reserva el stock
       ├──▶ Shipping Service    → crea el envío
       ├──▶ Notifications       → envía email de confirmación
       └──▶ Analytics Service   → registra la conversión

  Un evento → 5 reacciones independientes
  Orders no conoce a ninguno de los 5 consumidores
```

### Caso 2 — Notificación de eventos (Event Notification)

**Español:**
El producer simplemente notifica que algo ocurrió. Los consumers deciden qué hacer con esa información. El payload del evento es **mínimo** — solo lo esencial para que los consumers sepan qué ocurrió; si necesitan más datos, los consultan a su fuente.

**English:**
The producer simply notifies that something happened. Consumers decide what to do with that information. The event payload is **minimal** — just what's needed for consumers to know what happened; if they need more data, they query the source.

```
  Evento: user.registered
  Payload: { userId: "usr-8812", email: "...", timestamp: "..." }
       │
       ├──▶ Email Service       → envía email de bienvenida
       ├──▶ Loyalty Service     → crea cuenta de puntos (nivel Bronze)
       ├──▶ Analytics Service   → registra nuevo usuario
       └──▶ ML Service          → inicializa perfil de recomendaciones

  Cada consumer consulta sus propios datos adicionales si los necesita.
  El evento no incluye toda la info del usuario — solo la identidad.
```

### Caso 3 — Event-Carried State Transfer

**Español:**
Los consumers mantienen **réplicas locales** de datos de otros dominios, actualizadas mediante eventos. En lugar de consultar al servicio dueño en cada operación, el consumer tiene una copia local siempre actualizada. Reduce latencia y dependencias en tiempo de ejecución.

**English:**
Consumers maintain **local replicas** of data from other domains, updated via events. Instead of querying the owning service on every operation, the consumer has a locally updated copy. Reduces latency and runtime dependencies.

```
  Catalog Service publica:  product.price.updated
  Payload: { productId: "P-55", newPrice: 49.99, currency: "USD" }
       │
       └──▶ Orders Service mantiene tabla local:
            products_cache(productId, price, currency, updatedAt)

  Cuando Orders crea una orden:
    → NO llama a Catalog para obtener el precio
    → lee su propia tabla local (0ms de latencia de red)
    → el precio queda "frozen" en la orden en el momento correcto
```

### Caso 4 — Integración entre bounded contexts

**Español:**
Cuando dos dominios necesitan conocer cambios en el otro pero no deben acoplarse estructuralmente, los eventos son el contrato de integración. Cada dominio publica su propio lenguaje de eventos; el otro lo traduce a su modelo interno.

**English:**
When two domains need to know about changes in each other but must not be structurally coupled, events are the integration contract. Each domain publishes its own event language; the other translates it to its internal model.

```
  Catalog Domain          Event Bus          Search Domain
  ─────────────          ─────────          ────────────────
  Product(id,             product.           SearchDocument(
    name,          ──▶   updated   ──▶        id, title,
    category,      )                          tags, price,
    price)                                    searchableText
                                            )

  Catalog no sabe nada de Search.
  Search no sabe nada de la estructura interna de Catalog.
  El evento es el contrato de integración.
```

### Caso 5 — Procesamiento de streams en tiempo real

**Español:**
Cuando se necesita procesar un volumen alto y continuo de eventos (clicks, vistas, métricas, logs), EDA con un broker de alto throughput como Kafka permite procesar millones de eventos por segundo con múltiples consumidores especializados.

**English:**
When a high and continuous volume of events needs to be processed (clicks, views, metrics, logs), EDA with a high-throughput broker like Kafka allows processing millions of events per second with multiple specialized consumers.

```
  ShopFast — eventos de comportamiento de usuario
  ──────────────────────────────────────────────────
  user.product.viewed   ──▶ Kafka topic: user-behavior
  user.search.performed ──▶
  user.cart.added       ──▶
  user.checkout.started ──▶

  Consumers:
  ├──▶ ML Service           → actualiza modelo de recomendaciones en tiempo real
  ├──▶ Analytics Dashboard  → actualiza métricas de conversión (near real-time)
  └──▶ A/B Testing Service  → registra exposición y comportamiento por variante

  Volumen: ~50,000 eventos/segundo en peak (Black Friday)
```

### Caso 6 — Fire and Forget

**Español:**
El producer publica el evento y **no espera ninguna respuesta ni confirmación** del resultado. El flujo del producer continúa inmediatamente. No le importa si el evento fue procesado, cuándo ni por quién — solo le importa notificar que algo ocurrió.

Este patrón es ideal cuando el procesamiento downstream es un **side effect no crítico** para el flujo principal: si falla o se retrasa, el flujo principal no se ve afectado.

**English:**
The producer publishes the event and **does not wait for any response or confirmation** of the outcome. The producer's flow continues immediately. It does not care whether the event was processed, when, or by whom — it only cares about notifying that something happened.

This pattern is ideal when downstream processing is a **non-critical side effect** for the main flow: if it fails or is delayed, the main flow is unaffected.

```
  ShopFast — fire and forget en el flujo de orden

  Orders Service crea la orden y publica:
    ┌─────────────────────────────────────────┐
    │  publish("order.created", payload)      │
    │  return 201 Created  ◀── inmediato      │
    └─────────────────────────────────────────┘
         │
         │  Orders NO espera nada de aquí en adelante
         ▼
      Broker ──▶ Analytics Service (registra la conversión)
             ├──▶ ML Service       (actualiza modelo)
             └──▶ Loyalty Service  (suma puntos)

  Si Analytics falla hoy → la orden ya existe, el cliente ya pagó.
  El side effect se reintenta cuando Analytics vuelva online.
```

**Español:**
La clave del fire and forget no es que los eventos se pierdan (el broker los persiste), sino que el **producer es indiferente al resultado**. La responsabilidad del procesamiento recae completamente en el consumer.

**English:**
The key to fire and forget is not that events are lost (the broker persists them), but that the **producer is indifferent to the outcome**. The responsibility for processing falls entirely on the consumer.

### Caso 7 — Reliable Delivery (entrega confiable)

**Español:**
Cuando el evento representa una **acción de negocio crítica** — un pago, una reserva de inventario, una transferencia — no puede perderse bajo ninguna circunstancia, incluso si el consumer está momentáneamente caído. EDA con un broker persistente garantiza que el evento **sobrevive a fallos** de consumers, reinicios y caídas parciales del sistema.

**English:**
When the event represents a **critical business action** — a payment, an inventory reservation, a transfer — it cannot be lost under any circumstance, even if the consumer is momentarily down. EDA with a persistent broker guarantees that the event **survives failures** of consumers, restarts, and partial system outages.

```
┌─────────────────────────────────────────────────────────────────┐
│              RELIABLE DELIVERY — ShopFast                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Escenario: Payments Service cae durante 10 minutos             │
│                                                                 │
│  Sin EDA (síncrono):                                            │
│    Orders llama a Payments → Connection refused                 │
│    → la orden falla → el usuario ve un error                    │
│    → potencial pérdida de venta                                 │
│                                                                 │
│  Con EDA (event-driven + broker persistente):                   │
│    Orders publica order.created → Kafka persiste el evento      │
│    → Orders responde 202 Accepted al cliente (inmediato)        │
│    → Payments está caído: el evento espera en el topic          │
│    → 10 minutos después Payments vuelve online                  │
│    → Kafka entrega el evento → Payments procesa el cobro        │
│    → el usuario recibe confirmación de pago por email           │
│                                                                 │
│  El evento sobrevivió al fallo. El negocio no se interrumpió.  │
└─────────────────────────────────────────────────────────────────┘
```

**Español:**
La reliable delivery se apoya en la **durabilidad del broker**: Kafka persiste eventos en disco con replicación configurable (factor de replicación 3 es el estándar en producción). Los eventos no viven solo en memoria — sobreviven a reinicios del broker también.

**English:**
Reliable delivery relies on **broker durability**: Kafka persists events to disk with configurable replication (replication factor 3 is the production standard). Events do not live only in memory — they survive broker restarts as well.

### Caso 8 — Infinite Stream of Events

**Español:**
A diferencia del modelo request-response donde hay un inicio y un fin claros, EDA modela el sistema como un **flujo continuo e infinito de eventos** que nunca se detiene. El sistema no "termina" — procesa eventos mientras el negocio opera. Este modelo es especialmente poderoso para sistemas que necesitan reaccionar a comportamiento en tiempo real.

**English:**
Unlike the request-response model where there is a clear start and end, EDA models the system as a **continuous, infinite stream of events** that never stops. The system does not "finish" — it processes events as long as the business operates. This model is especially powerful for systems that need to react to behavior in real time.

```
  ShopFast — infinite stream of events (Kafka)
  ──────────────────────────────────────────────────────────────

  topic: user-behavior          topic: orders              topic: inventory
  ─────────────────────         ──────────────             ────────────────
  t=00:00  page.viewed          order.created              stock.updated
  t=00:01  product.viewed       order.paid                 stock.reserved
  t=00:02  search.performed     order.shipped              stock.low.alert
  t=00:03  cart.added           order.delivered            restock.triggered
  t=00:04  product.viewed       order.returned             ...
  t=00:05  checkout.started     ...
  ...      ...
  t=∞      (el stream no termina mientras ShopFast opera)

  Consumers siempre activos, procesando en tiempo real:
  ├──▶ ML Service        → actualiza recomendaciones con cada event
  ├──▶ Fraud Detection   → analiza patrones sobre ventana de tiempo
  └──▶ Warehouse System  → reabastece automáticamente cuando stock < umbral
```

**Español:**
La diferencia conceptual clave: en request-response, el sistema **reacciona a peticiones** del cliente. En el infinite stream model, el sistema **reacciona a hechos** que ocurren continuamente en el negocio — sin que nadie tenga que pedirle explícitamente que haga algo.

**English:**
The key conceptual difference: in request-response, the system **reacts to requests** from the client. In the infinite stream model, the system **reacts to facts** that continuously happen in the business — without anyone having to explicitly ask it to do anything.

### Caso 9 — Anomaly Detection / Pattern Recognition

**Español:**
EDA permite analizar el stream de eventos en tiempo real buscando **patrones anómalos** sobre una ventana de tiempo. A diferencia de las consultas síncronas (que analizan el estado actual), el reconocimiento de patrones analiza **secuencias de eventos** a lo largo del tiempo para detectar comportamientos que ningún evento individual revelaría por sí solo.

**English:**
EDA allows analyzing the event stream in real time looking for **anomalous patterns** over a time window. Unlike synchronous queries (which analyze current state), pattern recognition analyzes **sequences of events** over time to detect behaviors that no individual event would reveal on its own.

```
  ShopFast — Fraud Detection sobre stream de eventos

  Regla 1: >5 intentos de pago fallidos en 10 minutos
  ──────────────────────────────────────────────────────
  t=14:01  payment.failed  { userId: usr-99, card: *1234 }
  t=14:03  payment.failed  { userId: usr-99, card: *5678 }
  t=14:05  payment.failed  { userId: usr-99, card: *9012 }
  t=14:07  payment.failed  { userId: usr-99, card: *3456 }
  t=14:09  payment.failed  { userId: usr-99, card: *7890 }
                                    │
                          Fraud Detection Service
                          analiza ventana de 10 min
                                    │
                                    ▼
                          publica: fraud.alert.detected
                          { userId: usr-99, reason: "card_testing" }
                                    │
                          ├──▶ Account Service  → bloquea la cuenta
                          └──▶ Security Team   → alerta manual

  Regla 2: patrón de compra inusual
  ──────────────────────────────────────────────────────
  Mismo usuario compra 50 unidades del mismo producto
  desde una IP diferente a la habitual
  → ningún evento individual es sospechoso
  → la combinación de eventos en secuencia sí lo es
```

**Español:**
Este caso de uso es **imposible de implementar con request-response**: nadie hace una consulta para detectar fraude — el sistema debe observar el stream continuamente y reaccionar cuando el patrón emerge. El tiempo de detección es crítico: cuanto más rápido se detecta, menor es el daño.

**English:**
This use case is **impossible to implement with request-response**: no one makes a query to detect fraud — the system must continuously observe the stream and react when the pattern emerges. Detection time is critical: the faster it is detected, the smaller the damage.

### Caso 10 — Broadcasting

**Español:**
Broadcasting es el caso donde **un único evento debe llegar a todos los consumidores suscritos simultáneamente**, sin importar cuántos sean. Es el patrón pub/sub en su forma más pura: el producer publica una vez, y el broker distribuye el evento a todos los consumer groups registrados de forma independiente.

**English:**
Broadcasting is the case where **a single event must reach all subscribed consumers simultaneously**, regardless of how many there are. It is the pub/sub pattern in its purest form: the producer publishes once, and the broker distributes the event to all registered consumer groups independently.

```
  ShopFast — product.price.updated (broadcast a todos los dominios)

  Catalog Service publica:
    product.price.updated { productId: "P-55", newPrice: 39.99 }
                                        │
              ┌─────────────────────────┼──────────────────────────┐
              │                         │                          │
              ▼                         ▼                          ▼
      Orders Service            Search Service            Cart Service
   actualiza precio en        re-indexa el producto     actualiza precio
   órdenes pendientes         con nuevo precio en       en carritos activos
   (frozen correctamente)     resultados de búsqueda    que tienen P-55

              │                         │                          │
              ▼                         ▼                          ▼
      Recommendations           Analytics Service         Email Service
      Service actualiza         registra cambio           notifica a usuarios
      scoring con nuevo         de precio para            con alertas de
      precio                    reportes                  precio configuradas

  1 publicación → N consumers, cada uno con su lógica propia
  Añadir un nuevo consumer → 0 cambios en Catalog Service
```

**Español:**
El broadcasting es fundamentalmente diferente a hacer N llamadas síncronas: con request-response, Catalog tendría que conocer y llamar a cada uno de esos servicios. Con EDA, Catalog solo publica — la distribución es responsabilidad del broker. Añadir un nuevo consumer es **transparente para el producer**.

**English:**
Broadcasting is fundamentally different from making N synchronous calls: with request-response, Catalog would need to know and call each of those services. With EDA, Catalog only publishes — distribution is the broker's responsibility. Adding a new consumer is **transparent to the producer**.

### Caso 11 — Buffering

**Español:**
Cuando un producer genera eventos más rápido de lo que el consumer puede procesarlos, el broker actúa como **buffer** — absorbe la ráfaga de eventos y los entrega al consumer a su propio ritmo. Esto desacopla la velocidad de producción de la velocidad de consumo, evitando que un pico de tráfico derribe al consumer.

**English:**
When a producer generates events faster than the consumer can process them, the broker acts as a **buffer** — it absorbs the burst of events and delivers them to the consumer at its own pace. This decouples production speed from consumption speed, preventing a traffic spike from taking down the consumer.

```
  ShopFast — Black Friday: pico de órdenes

  Sin buffering (síncrono):
  ─────────────────────────────────────────────────────────────
  00:00 Black Friday comienza
  Orders: 8,000 req/seg  ──▶  Payments: capacidad 2,000 req/seg
                                        │
                                   SOBRECARGA
                                        │
                               Payments se cae ──▶ órdenes fallan
                               Los clientes ven errores 503


  Con buffering (EDA):
  ─────────────────────────────────────────────────────────────
  00:00 Black Friday comienza
  Orders: 8,000 eventos/seg ──▶ Kafka topic: orders
                                  │
                              Kafka acumula el surplus
                              (capacidad: millones de eventos)
                                  │
                              Payments: consume a 2,000/seg
                              ← procesa a su ritmo, sin caerse

  Resultado: todas las órdenes se procesan — con algo de retraso,
  pero sin pérdidas ni errores visibles al usuario.

  Consumer lag (eventos pendientes):
  00:00  lag = 0
  00:05  lag = 30,000  (pico de tráfico)
  00:10  lag = 25,000  (payments está procesando)
  00:20  lag = 10,000
  00:35  lag = 0       (payments se pone al día)
```

**Español:**
El buffering es uno de los beneficios más tangibles de EDA en producción. Sin él, cada pico de tráfico requiere sobreaprovisionar infraestructura para el peor caso. Con el broker como buffer, puedes dimensionar cada servicio para su **capacidad media**, no para el pico — reduciendo costos de infraestructura de forma significativa.

**English:**
Buffering is one of the most tangible benefits of EDA in production. Without it, every traffic spike requires over-provisioning infrastructure for the worst case. With the broker as a buffer, you can size each service for its **average capacity**, not the peak — significantly reducing infrastructure costs.

### Resumen — cuándo EDA es la elección correcta

```
┌─────────────────────────────────────────────────────────────────┐
│                USA EDA CUANDO...                                │
├─────────────────────────────────────────────────────────────────┤
│  ✔  Un evento desencadena reacciones en ≥2 servicios distintos  │
│  ✔  Los consumers pueden procesar con eventual consistency      │
│  ✔  Quieres añadir consumers futuros sin modificar el producer  │
│  ✔  Necesitas resiliencia: consumers pueden estar caídos        │
│     momentáneamente sin perder el evento (reliable delivery)    │
│  ✔  El producer no necesita saber el resultado (fire & forget)  │
│  ✔  El volumen de eventos es alto, continuo e infinito          │
│  ✔  Necesitas detectar patrones o anomalías en el stream        │
│  ✔  Un evento debe llegar a múltiples dominios (broadcasting)   │
│  ✔  El producer genera eventos más rápido de lo que el          │
│     consumer puede procesar (buffering para absorber picos)     │
│  ✔  Los consumers necesitan réplicas locales de datos           │
│     de otros dominios para operar de forma autónoma             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Casos del modelo Request-Response

**Español:**
El modelo request-response no desaparece en una arquitectura event-driven — sigue siendo la elección correcta para una categoría específica y muy importante de operaciones: aquellas donde el caller **necesita el resultado para continuar** o donde la consistencia fuerte e inmediata es un requisito no negociable.

**English:**
The request-response model does not disappear in an event-driven architecture — it remains the right choice for a specific and very important category of operations: those where the caller **needs the result to continue** or where strong, immediate consistency is a non-negotiable requirement.

### Caso 1 — Queries (lecturas de datos)

**Español:**
Las consultas de datos son el caso más claro para request-response. El usuario o el sistema pregunta algo y necesita la respuesta ahora para continuar. No hay "side effects" que propagar a múltiples dominios.

**English:**
Data queries are the clearest case for request-response. The user or system asks something and needs the answer now to continue. There are no side effects to propagate to multiple domains.

```
  ShopFast — ejemplos de queries síncronas

  GET /api/products/P-55          → Catalog Service devuelve datos del producto
  GET /api/orders/ord-00421       → Orders Service devuelve estado de la orden
  GET /api/cart                   → Cart Service devuelve contenido del carrito
  GET /api/search?q=zapatillas    → Search Service devuelve resultados paginados

  En todos estos casos:
  • El cliente necesita la respuesta para renderizar la UI
  • No hay consumers downstream que necesiten reaccionar
  • La respuesta debe ser inmediata (<200ms)
```

### Caso 2 — Operaciones que requieren confirmación inmediata

**Español:**
Cuando el resultado de la operación debe conocerse en el mismo flujo (login, verificación de stock antes de checkout, validación de cupón), request-response es la única opción válida. El usuario no puede continuar hasta saber si la operación fue exitosa.

**English:**
When the result of an operation must be known within the same flow (login, stock check before checkout, coupon validation), request-response is the only valid option. The user cannot continue until they know whether the operation succeeded.

```
  ShopFast — operaciones síncronas obligatorias

  POST /api/auth/login
    → el cliente necesita el JWT para hacer cualquier otra petición
    → no puede continuar sin la respuesta

  GET /api/inventory/check?productId=P-55&qty=2
    → antes de mostrar "Añadir al carrito", verificar disponibilidad
    → si el stock está agotado, no tiene sentido continuar el flujo

  POST /api/coupons/validate
    → el usuario ingresó un cupón y espera ver el descuento ahora
    → consistencia fuerte requerida (el cupón puede usarse solo una vez)
```

### Caso 3 — Llamadas a APIs externas

**Español:**
Las APIs de terceros (pasarelas de pago, carriers de envío, servicios de fraude) son síncronas por naturaleza — no consumen eventos del broker interno. La comunicación con ellas siempre es request-response, aunque el flujo interno sea event-driven.

**English:**
Third-party APIs (payment gateways, shipping carriers, fraud services) are synchronous by nature — they do not consume events from the internal broker. Communication with them is always request-response, even if the internal flow is event-driven.

```
  ShopFast — Payments Service (interno event-driven, externo síncrono)

  Payments Service consume: order.created (event-driven) ✔
       │
       └──▶ Stripe API: POST /charges  (request-response síncrono) ✔
                 │
                 └── respuesta: { chargeId, status: "succeeded" }
                       │
                       └──▶ publica: payment.processed (event-driven) ✔

  El broker interno y la API externa son dos mundos distintos.
  La frontera con el mundo externo siempre es síncrona.
```

### Resumen — cuándo Request-Response es la elección correcta

```
┌─────────────────────────────────────────────────────────────────┐
│             USA REQUEST-RESPONSE CUANDO...                      │
├─────────────────────────────────────────────────────────────────┤
│  ✔  Es una lectura (query) — no hay side effects                │
│  ✔  El caller necesita la respuesta para continuar el flujo     │
│  ✔  Se requiere consistencia fuerte e inmediata                 │
│  ✔  El flujo involucra solo 2 partes (caller + callee)          │
│  ✔  La operación es con una API externa de terceros             │
│  ✔  El resultado determina si el flujo continúa o se detiene    │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "A good architecture uses both models. Events for state changes with multiple downstream reactions. Request-response for queries and operations where the caller needs an immediate answer to proceed."

---

## 3. Patrones de entrega de eventos

**Español:**
No todos los eventos tienen los mismos requisitos de entrega. Un evento de click de usuario puede perderse sin consecuencias. Un evento de pago procesado no puede perderse ni duplicarse. El patrón de entrega define el **contrato de fiabilidad** entre el broker y los consumers.

**English:**
Not all events have the same delivery requirements. A user click event can be lost without consequences. A payment processed event cannot be lost or duplicated. The delivery pattern defines the **reliability contract** between the broker and consumers.

### Patrón 1 — At-Most-Once (fire and forget)

**Español:**
El evento se envía **una sola vez**. Si el consumer está caído o el mensaje se pierde en tránsito, no hay reintento. Es el patrón más simple y de mayor throughput, pero sin garantías de entrega.

**English:**
The event is sent **once only**. If the consumer is down or the message is lost in transit, there is no retry. It is the simplest and highest-throughput pattern, but with no delivery guarantees.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AT-MOST-ONCE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer ──▶ Broker ──▶ Consumer                              │
│                  │                                              │
│              (si el mensaje                                     │
│               se pierde aquí,                                   │
│               no hay reintento)                                 │
│                                                                 │
│  Garantía:     0 o 1 entrega                                    │
│  Duplicados:   Imposibles                                       │
│  Pérdida:      Posible                                          │
│  Complejidad:  Muy baja                                         │
│  Throughput:   Muy alto                                         │
│                                                                 │
│  Casos de uso en ShopFast:                                      │
│  • Eventos de clicks / page views (analytics)                   │
│  • Métricas de performance (latencia, CPU)                      │
│  • Logs de actividad de bajo valor                              │
│  → Perder 1 en 10,000 no impacta el negocio                    │
└─────────────────────────────────────────────────────────────────┘
```

### Patrón 2 — At-Least-Once (con reintentos)

**Español:**
El broker **reintenta la entrega** hasta recibir un acknowledgment (ACK) del consumer. Garantiza que el mensaje llegue, pero si el consumer falla **después de procesar** y antes de enviar el ACK, el mensaje se reenvía y el consumer lo procesa dos veces. El consumer **debe ser idempotente**.

**English:**
The broker **retries delivery** until it receives an acknowledgment (ACK) from the consumer. It guarantees the message arrives, but if the consumer fails **after processing** and before sending the ACK, the message is resent and the consumer processes it twice. The consumer **must be idempotent**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    AT-LEAST-ONCE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer ──▶ Broker ──▶ Consumer                              │
│                  │            │                                 │
│              persiste         │ procesa + ACK                   │
│              el evento        │                                 │
│                  │            │ si falla antes del ACK:         │
│                  └──────────▶ reenvía                           │
│                                                                 │
│  Garantía:     ≥1 entrega (puede ser más de una)                │
│  Duplicados:   Posibles → consumer debe ser IDEMPOTENTE         │
│  Pérdida:      Imposible (mientras el broker esté disponible)   │
│  Complejidad:  Media                                            │
│                                                                 │
│  Idempotencia en ShopFast:                                      │
│  Payments Service recibe order.created dos veces:               │
│    → 1ª vez: INSERT INTO payments(orderId=421) → OK             │
│    → 2ª vez: INSERT INTO payments(orderId=421)                  │
│              → UNIQUE CONSTRAINT violation → ignora el evento   │
│  El resultado es el mismo: un solo cargo.                       │
│                                                                 │
│  Casos de uso: order.created, payment.processed,                │
│  inventory.reserved — cualquier evento de negocio crítico       │
└─────────────────────────────────────────────────────────────────┘
```

### Patrón 3 — Exactly-Once

**Español:**
El evento se entrega y procesa **exactamente una vez**, sin pérdidas ni duplicados. Es el patrón más difícil de implementar — requiere coordinación transaccional entre el producer, el broker y el consumer. Kafka lo soporta con **idempotent producers** + **transactional APIs**. En la práctica, la mayoría de los sistemas de alta disponibilidad usan at-least-once + consumidores idempotentes, que logran el mismo resultado con menos complejidad.

**English:**
The event is delivered and processed **exactly once**, with no losses or duplicates. It is the hardest pattern to implement — it requires transactional coordination among the producer, broker, and consumer. Kafka supports it with **idempotent producers** + **transactional APIs**. In practice, most high-availability systems use at-least-once + idempotent consumers, which achieve the same result with less complexity.

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXACTLY-ONCE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Requiere:                                                      │
│  • Idempotent producer (Kafka: enable.idempotence=true)         │
│  • Transactional producer (Kafka transactions API)              │
│  • Consumer que procesa dentro de la misma transacción          │
│                                                                 │
│  Garantía:     exactamente 1 entrega y 1 procesamiento          │
│  Duplicados:   Imposibles                                       │
│  Pérdida:      Imposible                                        │
│  Complejidad:  Alta — overhead de latencia y configuración      │
│  Throughput:   Reducido (transacciones distribuidas son lentas) │
│                                                                 │
│  Casos de uso en ShopFast:                                      │
│  • Transferencias financieras entre cuentas                     │
│  • Actualizaciones de saldo de loyalty points                   │
│  • Deducción de inventario (overselling es inaceptable)         │
│                                                                 │
│  Alternativa práctica: at-least-once + idempotency key          │
│  → mismo resultado, menor complejidad operacional               │
└─────────────────────────────────────────────────────────────────┘
```

### Pub/Sub vs Point-to-Point (Queue)

**Español:**
Además de las garantías de entrega, hay dos topologías fundamentales que definen **cuántos consumers reciben cada mensaje**.

**English:**
Beyond delivery guarantees, there are two fundamental topologies that define **how many consumers receive each message**.

```
  PUB/SUB — todos los subscribers reciben una copia
  ─────────────────────────────────────────────────────────────
  order.created ──▶ Kafka topic
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          Payments   Inventory  Shipping
         (consumer   (consumer  (consumer
          group A)    group B)   group C)

  Cada consumer group recibe su propia copia del evento.
  Uso: notificación a múltiples dominios independientes.


  POINT-TO-POINT (Queue) — solo un consumer recibe cada mensaje
  ─────────────────────────────────────────────────────────────
  order.created ──▶ Queue
                      │
          ┌───────────┼───────────┐
          ▼                       ▼
      Payments-1              Payments-2
      (instancia)             (instancia)

  Solo UNA instancia procesa cada mensaje (competing consumers).
  Uso: escalar horizontalmente el procesamiento de una tarea.
  Ej: 3 instancias de Payments compiten por los mensajes de la queue
      → throughput 3x, cada orden se procesa exactamente una vez.
```

### Event Streaming

**Español:**
Event Streaming es un patrón de entrega donde los eventos **no se eliminan del broker después de ser consumidos** — se retienen durante un período configurable (días, semanas, o indefinidamente). Cualquier consumer puede leer el stream desde cualquier posición en el tiempo: desde el inicio, desde un offset específico, o solo los nuevos eventos. El broker actúa como un **log de eventos ordenado e inmutable**.

**English:**
Event Streaming is a delivery pattern where events **are not deleted from the broker after being consumed** — they are retained for a configurable period (days, weeks, or indefinitely). Any consumer can read the stream from any position in time: from the beginning, from a specific offset, or only new events. The broker acts as an **ordered, immutable event log**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT STREAMING (Kafka)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Topic: orders  (retención: 7 días)                             │
│  ──────────────────────────────────────────────────────────     │
│  offset:  0        1        2        3        4        5        │
│         [evt]    [evt]    [evt]    [evt]    [evt]    [evt] ──▶  │
│                                                                 │
│  Consumer A (Payments):     lee desde offset actual → en vivo   │
│  Consumer B (Analytics):    lee desde offset 0 → replay total   │
│  Consumer C (nuevo servicio que se incorpora hoy):              │
│                             lee desde offset 3 → desde ayer     │
│                                                                 │
│  El evento NO desaparece cuando A lo consume.                   │
│  B y C lo leen independientemente, en su propio tiempo.         │
└─────────────────────────────────────────────────────────────────┘
```

**Español:**
La diferencia fundamental con una queue tradicional (point-to-point):

**English:**
The fundamental difference from a traditional queue (point-to-point):

```
  QUEUE tradicional                   EVENT STREAMING
  ──────────────────────              ──────────────────────────────
  Mensaje publicado                   Evento publicado
       │                                   │
       ▼                                   ▼
  Consumer lo lee                    Kafka lo persiste en el log
       │                                   │
       ▼                              ┌────┴──────────────────────┐
  Mensaje ELIMINADO                   │  Consumer A lee offset 5  │
  de la queue                         │  Consumer B lee offset 2  │
                                      │  Consumer C lee offset 0  │
  Solo 1 consumer                     │  (cada uno avanza a su    │
  puede leerlo.                       │   propio ritmo)           │
                                      └───────────────────────────┘
                                      El evento permanece hasta
                                      que expira la retención.
```

### Casos de uso del Event Streaming en ShopFast

```
┌─────────────────────────────────────────────────────────────────┐
│  1. REPLAY para nuevos servicios                                │
│     Loyalty Service se incorpora 6 meses después del           │
│     lanzamiento → reproduce todos los order.created desde       │
│     el inicio → calcula el saldo de puntos correcto para        │
│     cada usuario sin necesidad de migración de datos            │
│                                                                 │
│  2. REPLAY para recuperación de fallos                          │
│     Analytics Service corrompe su base de datos → reproduce     │
│     el stream de los últimos 7 días → reconstruye todas         │
│     las métricas sin pérdida de información                     │
│                                                                 │
│  3. MÚLTIPLES consumers a diferente velocidad                   │
│     Real-time dashboard: consume al instante (lag ~0)           │
│     ML training pipeline: consume en batch cada noche           │
│     Compliance audit: consume semanalmente para reportes        │
│     Los tres leen el MISMO topic sin interferirse               │
│                                                                 │
│  4. AUDITORÍA y trazabilidad                                    │
│     El stream es el registro completo de todo lo que            │
│     ocurrió en el sistema — inmutable y ordenado por tiempo.    │
│     "¿Qué pasó exactamente con la orden 00421?"                 │
│     → reproduce sus eventos en orden cronológico                │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "With event streaming, the broker is not just a pipe — it is the system of record. The stream is the truth. Any service can derive its own view of reality by reading the stream from any point in time."

### Tabla comparativa de patrones de entrega

| Patrón | Garantía | Duplicados | Pérdida | Complejidad | Casos de uso |
|--------|----------|-----------|---------|-------------|-------------|
| **At-most-once** | 0 o 1 entrega | No | Posible | Muy baja | Analytics, métricas, logs no críticos |
| **At-least-once** | ≥1 entrega | Posibles | No | Media | Eventos de negocio — requiere idempotencia |
| **Exactly-once** | Exactamente 1 | No | No | Alta | Transacciones financieras, inventario crítico |
| **Pub/Sub** | Todos los groups | — | — | Baja | Fan-out a múltiples dominios independientes |
| **Point-to-Point** | Solo 1 consumer | — | — | Baja | Escalar procesamiento horizontal (competing consumers) |
| **Event Streaming** | Persistencia configurable | — | No (hasta expirar retención) | Media | Replay, auditoría, múltiples consumers a diferente velocidad |

> **Regla de Pogrebinsky:** "In practice, at-least-once delivery with idempotent consumers is the sweet spot — it gives you the reliability of exactly-once with far less operational complexity. Design your consumers to be idempotent first, then choose your delivery guarantee."
