# Introduction to Event-Driven Architecture

---

## 1. Motivación para la Arquitectura Orientada a Eventos

**Español:**
En una arquitectura de microservicios con comunicación síncrona (REST/gRPC), cada servicio que necesita datos de otro lo llama directamente y **espera** la respuesta. Esto funciona bien para operaciones simples, pero en flujos de negocio complejos que involucran múltiples servicios se convierte en una fuente de acoplamiento temporal, fragilidad y latencia acumulada.

**English:**
In a microservices architecture with synchronous communication (REST/gRPC), every service that needs data from another calls it directly and **waits** for the response. This works fine for simple operations, but in complex business flows involving multiple services it becomes a source of temporal coupling, fragility, and accumulated latency.

### ShopFast — flujo de "Orden creada" con comunicación síncrona

```
  Cliente
     │
     ▼
 Orders Service
     │
     ├──▶ Payments Service  ──▶ (espera respuesta)
     │         │
     │         └──▶ Inventory Service  ──▶ (espera respuesta)
     │                   │
     │                   └──▶ Shipping Service  ──▶ (espera respuesta)
     │                               │
     │                               └──▶ Notifications Service  ──▶ (espera respuesta)
     │
     └── responde al cliente (después de que TODOS terminaron)
```

### Consecuencias del acoplamiento síncrono en cadena

```
┌─────────────────────────────────────────────────────────────────┐
│             PROBLEMAS DEL MODELO SÍNCRONO EN CADENA             │
├─────────────────────────────────────────────────────────────────┤
│  1. Acoplamiento temporal: Orders conoce la existencia de       │
│     Payments, Inventory, Shipping y Notifications. Cualquier    │
│     cambio en sus APIs afecta a Orders directamente.            │
│                                                                 │
│  2. Latencia acumulada: si cada servicio tarda 50ms,            │
│     4 llamadas en serie = 200ms mínimo. En producción con       │
│     reintentos puede superar 1 segundo.                         │
│                                                                 │
│  3. Fragilidad en cascada: si Notifications falla, la           │
│     orden completa falla — aunque el pago ya se procesó.        │
│                                                                 │
│  4. Escalabilidad acoplada: para escalar Orders hay que         │
│     escalar también todos los servicios downstream que          │
│     llama síncronamente.                                        │
│                                                                 │
│  5. Disponibilidad reducida: la disponibilidad total del        │
│     flujo = 99.9% × 99.9% × 99.9% × 99.9% = 99.6%.              │
│     Cada servicio adicional en la cadena la degrada.            │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "In synchronous chains, the weakest service in the chain determines the reliability of the entire flow. Event-driven architecture breaks that chain."

---

## 2. Concepto fundamental de la Arquitectura Orientada a Eventos

**Español:**
En la Arquitectura Orientada a Eventos (EDA), los servicios no se llaman entre sí directamente. En cambio, cuando algo importante sucede en el sistema, el servicio responsable **publica un evento** en un canal compartido (message broker). Cualquier servicio interesado en ese evento lo **consume de forma independiente**, sin que el publicador sepa quién lo va a leer ni cuándo.

**English:**
In Event-Driven Architecture (EDA), services do not call each other directly. Instead, when something significant happens in the system, the responsible service **publishes an event** to a shared channel (message broker). Any service interested in that event **consumes it independently**, without the publisher knowing who will read it or when.

### Los tres actores fundamentales

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ┌─────────────┐     publica     ┌──────────────────────┐       │
│  │  PRODUCER   │─────────────▶  │    MESSAGE BROKER    │       │
│  │  (Publisher)│                 │                      │       │
│  └─────────────┘                 │  • Kafka             │       │
│                                  │  • RabbitMQ          │       │
│  Conoce el evento.               │  • AWS SNS/SQS       │       │
│  NO conoce a los                 │  • Azure Service Bus │       │
│  consumidores.                   │                      │       │
│                                  └──────────┬───────────┘       │
│                                             │  consume          │
│                                   ┌─────────┼─────────┐         │
│                                   │         │         │         │
│                                   ▼         ▼         ▼         │
│                              ┌─────────┐ ┌──────┐ ┌──────┐      │
│                              │CONSUMER │ │  C2  │ │  C3  │      │
│                              └─────────┘ └──────┘ └──────┘      │
│                                                                 │
│  Los consumidores son independientes entre sí y del producer.   │
└─────────────────────────────────────────────────────────────────┘
```

### Anatomía de un evento

**Español:**
Un evento es un **registro inmutable de algo que ya ocurrió** en el sistema. No es una instrucción ("haz esto"), es un hecho ("esto sucedió"). Esta distinción es fundamental: el producer no manda, notifica.

**English:**
An event is an **immutable record of something that already happened** in the system. It is not an instruction ("do this"), it is a fact ("this happened"). This distinction is fundamental: the producer does not command, it notifies.

```
┌─────────────────────────────────────────────────────────────────┐
│                   EVENTO: order.created                         │
├─────────────────────────────────────────────────────────────────┤
│  eventId:    "evt-9f3a2c1d"                                     │
│  eventType:  "order.created"                                    │
│  timestamp:  "2026-05-22T14:32:11Z"                             │
│  source:     "orders-service"                                   │
│  version:    "1.0"                                              │
│  payload:                                                       │
│    orderId:   "ord-00421"                                       │
│    userId:    "usr-8812"                                        │
│    items:     [{ productId: "P-55", qty: 2, price: 49.99 }]    │
│    total:     99.98                                             │
│    currency:  "USD"                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Propiedades clave de un evento

| Propiedad | Descripción |
|-----------|-------------|
| **Inmutabilidad** | Un evento no se modifica una vez publicado. Representa el pasado. |
| **Asincronía** | El producer no espera a que nadie lo consuma. Publica y continúa. |
| **Desacoplamiento** | El producer no sabe quién consume ni cuántos consumidores hay. |
| **Durabilidad** | El broker persiste el evento; los consumers lo leen cuando estén disponibles. |
| **Replay** | Los eventos pueden reproducirse para reconstruir estado o depurar. |

> **Regla de Pogrebinsky:** "An event is a fact, not a command. 'OrderCreated' is correct. 'ProcessPayment' is a command — it belongs in synchronous RPC, not in an event bus."

---

## 3. Modelo Request-Response vs Modelo Event-Driven

**Español:**
Estos dos modelos no son opuestos excluyentes — en una arquitectura real conviven. La clave está en saber cuándo usar cada uno según las necesidades de consistencia, latencia y acoplamiento del flujo.

**English:**
These two models are not mutually exclusive opposites — in a real architecture they coexist. The key is knowing when to use each one based on the consistency, latency, and coupling needs of the flow.

### Comparación estructural

```
  REQUEST-RESPONSE (síncrono)
  ─────────────────────────────────────────────────────
  Client ──▶ Service A ──▶ Service B ──▶ Service C
               ◀────────── ◀────────── ◀──────────
  Caller espera. Caller sabe el resultado inmediatamente.
  Acoplamiento fuerte en tiempo y API.


  EVENT-DRIVEN (asíncrono)
  ─────────────────────────────────────────────────────
  Service A ──▶ [order.created] ──▶ Message Broker
                                         │
                              ┌──────────┼──────────┐
                              ▼          ▼          ▼
                          Service B  Service C  Service D
  Publisher no espera. No sabe quién consume ni cuándo.
  Desacoplamiento total en tiempo y API.
```

### Tabla comparativa

| Dimensión | Request-Response | Event-Driven |
|-----------|-----------------|--------------|
| **Flujo** | Síncrono — caller espera respuesta | Asíncrono — producer no espera |
| **Acoplamiento** | Fuerte — A conoce la API de B | Débil — A solo conoce el esquema del evento |
| **Consistencia** | Fuerte — resultado inmediato y conocido | Eventual — los consumidores procesan con retraso |
| **Latencia** | Acumulada en cadena | Independiente por consumidor |
| **Disponibilidad** | Degradada si cualquier eslabón falla | Alta — broker persiste eventos aunque consumers estén caídos |
| **Escalabilidad** | Acoplada — escalar A puede requerir escalar B | Independiente — cada consumer escala por su cuenta |
| **Visibilidad del resultado** | Inmediata — respuesta en la misma request | Diferida — requiere polling, webhooks o long polling |
| **Debugging** | Simple — stack trace directo | Complejo — requiere tracing distribuido |
| **Cuándo usarlo** | Lectura de datos, operaciones que requieren respuesta inmediata | Notificaciones, flujos multi-servicio, side effects |

### Cuándo usar cada modelo — regla práctica

```
┌─────────────────────────────────────────────────────────────────┐
│                    USA REQUEST-RESPONSE cuando...               │
├─────────────────────────────────────────────────────────────────┤
│  • El cliente necesita la respuesta para continuar              │
│    (ej: "¿está este producto en stock?" antes de añadir al cart)│
│  • La operación es una lectura de datos                         │
│  • Necesitas consistencia fuerte e inmediata                    │
│  • El flujo involucra solo 2 servicios                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    USA EVENT-DRIVEN cuando...                   │
├─────────────────────────────────────────────────────────────────┤
│  • La operación desencadena side effects en múltiples servicios │
│    (ej: orden creada → pago + inventario + envío + notificación)│
│  • Los consumidores pueden procesar con retraso (eventual)      │
│  • Quieres añadir nuevos consumidores sin cambiar el producer   │
│  • Necesitas resiliencia: si Notifications falla, la orden      │
│    ya fue confirmada — el evento se reintenta después            │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Not everything should be event-driven. Use synchronous calls for queries and operations that need immediate feedback. Use events for state changes that others need to react to."

### Síncrono vs Asíncrono — la diferencia fundamental

**Español:**
La distinción síncrono/asíncrono no es solo sobre velocidad — es sobre **quién controla el tiempo de ejecución**. En el modelo síncrono, el caller bloquea su propio hilo de ejecución hasta recibir respuesta. En el modelo asíncrono, el producer continúa ejecutando inmediatamente; el procesamiento ocurre en paralelo en otro contexto.

**English:**
The synchronous/asynchronous distinction is not just about speed — it is about **who controls execution time**. In the synchronous model, the caller blocks its own execution thread until a response arrives. In the asynchronous model, the producer continues executing immediately; processing happens in parallel in a different context.

```
  SÍNCRONO — hilo del caller bloqueado
  ──────────────────────────────────────────────────────────────
  Orders  │████ publica ████│────── espera ──────│████ continúa ████│
             │                                   │
  Payments   │              │████ procesa ████│
             t0             t1                t2

  El caller está bloqueado entre t1 y t2. No puede hacer nada más.


  ASÍNCRONO — hilo del caller libre
  ──────────────────────────────────────────────────────────────
  Orders  │████ publica ████│████████████████████ continúa ████│
             │
  Payments   │              ████ procesa (independientemente) ████
             t0

  El caller publica en t0 y sigue ejecutando de inmediato.
  Payments procesa cuando puede, sin bloquear a Orders.
```

**Español:**
Esta diferencia tiene consecuencias directas en la **resiliencia**: si Payments tarda 2 segundos, en el modelo síncrono Orders también tarda 2 segundos. En el modelo asíncrono, Orders responde al cliente en milisegundos y Payments procesa en background — el usuario no percibe la latencia del procesamiento interno.

**English:**
This difference has direct consequences for **resilience**: if Payments takes 2 seconds, in the synchronous model Orders also takes 2 seconds. In the asynchronous model, Orders responds to the client in milliseconds and Payments processes in the background — the user does not perceive the latency of internal processing.

### Inversion of Control

**Español:**
En el modelo request-response, **Service A controla el flujo**: decide explícitamente qué servicios llamar, en qué orden, y qué hacer con cada respuesta. Cuando se añade un nuevo servicio, hay que modificar A para que lo llame. El control del flujo está centralizado en el caller.

En el modelo event-driven, ese control se **invierte**: Service A publica un evento y deja de tener control sobre qué ocurre a continuación. Son los **consumers quienes deciden** si reaccionan al evento y cómo. A no sabe que B, C o D existen.

**English:**
In the request-response model, **Service A controls the flow**: it explicitly decides which services to call, in what order, and what to do with each response. When a new service is added, A must be modified to call it. Flow control is centralized in the caller.

In the event-driven model, that control is **inverted**: Service A publishes an event and loses control over what happens next. It is the **consumers who decide** whether to react to the event and how. A does not know that B, C, or D exist.

```
  REQUEST-RESPONSE — control centralizado en el caller
  ─────────────────────────────────────────────────────────────
  Orders Service (el que manda):
    1. call Payments.charge()
    2. call Inventory.reserve()
    3. call Shipping.create()
    4. call Notifications.send()

  Para añadir Loyalty Points → hay que editar Orders.


  EVENT-DRIVEN — control invertido (Inversion of Control)
  ─────────────────────────────────────────────────────────────
  Orders Service (solo notifica):
    1. publish("order.created", payload)
    → fin. Orders no sabe qué pasa después.

  Payments   → decide reaccionar: procesa el cobro
  Inventory  → decide reaccionar: reserva el stock
  Shipping   → decide reaccionar: crea el envío
  Loyalty    → decide reaccionar: suma puntos  ← NUEVO, 0 cambios en Orders
```

**Español:**
La Inversion of Control en EDA es el mismo principio que en Dependency Injection aplicado a nivel de arquitectura: en lugar de que el componente de alto nivel controle a los de bajo nivel, **los componentes de bajo nivel se registran y reaccionan de forma autónoma**.

**English:**
Inversion of Control in EDA is the same principle as Dependency Injection applied at the architecture level: instead of the high-level component controlling the low-level ones, **the low-level components register themselves and react autonomously**.

### Loose Coupling

**Español:**
El loose coupling (acoplamiento débil) es el beneficio arquitectónico más importante de EDA. En el modelo síncrono, los servicios están fuertemente acoplados en tres dimensiones. EDA elimina ese acoplamiento en las tres.

**English:**
Loose coupling is the most important architectural benefit of EDA. In the synchronous model, services are tightly coupled in three dimensions. EDA eliminates that coupling in all three.

```
┌─────────────────────────────────────────────────────────────────┐
│                  LOS 3 TIPOS DE ACOPLAMIENTO                    │
├─────────────────┬───────────────────────┬───────────────────────┤
│                 │  REQUEST-RESPONSE     │  EVENT-DRIVEN         │
├─────────────────┼───────────────────────┼───────────────────────┤
│ Acoplamiento    │ FUERTE: A debe estar  │ DÉBIL: A publica y    │
│ temporal        │ up cuando llama a B.  │ continúa. B puede     │
│                 │ Si B está caído, A    │ procesar cuando       │
│                 │ falla.                │ vuelva online.        │
├─────────────────┼───────────────────────┼───────────────────────┤
│ Acoplamiento    │ FUERTE: A conoce la   │ DÉBIL: A solo conoce  │
│ de identidad    │ URL, el nombre y la   │ el nombre del evento  │
│                 │ API de B. Cambiar B   │ y su schema. B puede  │
│                 │ afecta a A.           │ cambiar internamente  │
│                 │                       │ sin afectar a A.      │
├─────────────────┼───────────────────────┼───────────────────────┤
│ Acoplamiento    │ FUERTE: A decide      │ DÉBIL: A no sabe      │
│ de flujo        │ quién reacciona y en  │ quién consume ni      │
│                 │ qué orden.            │ cuántos consumers     │
│                 │                       │ hay. El broker        │
│                 │                       │ gestiona la entrega.  │
└─────────────────┴───────────────────────┴───────────────────────┘
```

**Español:**
En ShopFast: Orders Service no conoce a Payments, Inventory, Shipping ni Notifications. Solo conoce la definición del evento `order.created`. Cada uno de esos servicios puede desplegarse, escalarse, reescribirse o eliminarse **sin que Orders sepa ni le importe**. Eso es loose coupling en la práctica.

**English:**
In ShopFast: Orders Service does not know Payments, Inventory, Shipping, or Notifications. It only knows the definition of the `order.created` event. Each of those services can be deployed, scaled, rewritten, or removed **without Orders knowing or caring**. That is loose coupling in practice.

> **Regla de Pogrebinsky:** "Loose coupling is not a nice-to-have — it is the reason microservices exist. Event-driven architecture is the mechanism that achieves it at scale."

---

# Event-Driven Architecture — Real-life Example

**Español:**
El ejemplo más claro de EDA en e-commerce es el flujo completo de creación de una orden. En ShopFast, cuando un usuario confirma su compra, **un único evento desencadena de forma independiente y asíncrona** el procesamiento en cuatro dominios distintos — sin que Orders Service sepa quién reacciona ni en qué orden.

**English:**
The clearest example of EDA in e-commerce is the complete order creation flow. In ShopFast, when a user confirms their purchase, **a single event independently and asynchronously triggers** processing across four different domains — without Orders Service knowing who reacts or in what order.

### Flujo completo: order.created en ShopFast

```
  Usuario confirma compra
         │
         ▼
  ┌──────────────┐
  │Orders Service│  ──── publica ────▶  [ order.created ]
  └──────────────┘                            │
    (responde 202                    ┌─────────┼──────────────────────┐
     Accepted al                     │         │          │           │
     cliente de                      ▼         ▼          ▼           ▼
     inmediato)               Payments    Inventory   Shipping    Notifications
                               Service     Service     Service      Service
                                  │            │           │            │
                                  │            │           │            │
                           charge card   reserve      create        send
                           → publica     stock        shipment      email/push
                           payment.      → publica    → publica     → (terminal,
                           processed     inventory.   shipping.      no evento)
                                         reserved     created
```

**Español:**
Cada servicio consumer es **completamente independiente**. Si Notifications Service está caído en el momento en que se publica el evento, el broker retiene el evento y Notifications lo consumirá cuando vuelva a estar disponible. La orden ya fue confirmada y el pago ya fue procesado — la caída de Notifications no afecta al flujo principal.

**English:**
Each consumer service is **completely independent**. If Notifications Service is down when the event is published, the broker retains the event and Notifications will consume it when it comes back online. The order was already confirmed and payment was already processed — a Notifications outage does not affect the main flow.

### Cadena de eventos secundarios

**Español:**
Los consumidores del evento inicial pueden a su vez publicar nuevos eventos, creando una **cadena de eventos reactiva** sin coordinación centralizada.

**English:**
The consumers of the initial event can in turn publish new events, creating a **reactive event chain** without centralized coordination.

```
  order.created
       │
       ├──▶ Payments Service  ──▶  payment.processed
       │                                │
       │                                └──▶ Fraud Detection Service  ──▶  fraud.cleared
       │
       ├──▶ Inventory Service  ──▶  inventory.reserved
       │                                │
       │                                └──▶ Warehouse Service  ──▶  picking.initiated
       │
       └──▶ Shipping Service  ──▶  shipping.created
                                        │
                                        └──▶ Carrier API  ──▶  (llamada síncrona externa)
```

### Beneficios demostrados en el ejemplo

```
┌─────────────────────────────────────────────────────────────────┐
│              BENEFICIOS OBSERVADOS EN EL FLUJO                  │
├─────────────────────────────────────────────────────────────────┤
│  1. Desacoplamiento: Orders no importa Payments, Inventory,     │
│     Shipping ni Notifications. Si mañana se añade un nuevo      │
│     servicio de Loyalty Points, solo se subscribe al evento     │
│     order.created — Orders no cambia una línea de código.       │
│                                                                 │
│  2. Resiliencia: cada servicio falla de forma independiente.    │
│     Un bug en Notifications no bloquea el pago ni el envío.     │
│                                                                 │
│  3. Escalabilidad independiente: durante el Black Friday,       │
│     Shipping Service puede tener 20 instancias consumiendo el   │
│     mismo topic mientras Orders tiene solo 3.                   │
│                                                                 │
│  4. Respuesta inmediata al cliente: Orders devuelve 202         │
│     Accepted en <50ms. El resto ocurre en background.           │
│     El cliente no espera a que se procese el pago.              │
│                                                                 │
│  5. Auditabilidad: el broker retiene todos los eventos.         │
│     Es posible reconstruir el estado de cualquier orden         │
│     reproduciendo su secuencia de eventos (event sourcing).     │
└─────────────────────────────────────────────────────────────────┘
```

### Añadir un nuevo consumidor sin tocar el producer

```
  ANTES (sin Loyalty Points):                DESPUÉS (con Loyalty Points):

  order.created ──▶ Payments                 order.created ──▶ Payments
                ├──▶ Inventory                             ├──▶ Inventory
                ├──▶ Shipping                              ├──▶ Shipping
                └──▶ Notifications                         ├──▶ Notifications
                                                           └──▶ Loyalty Points  ← NUEVO
                                                                (0 cambios en
                                                                 Orders Service)
```

**Español:**
Este es el poder real del desacoplamiento orientado a eventos: el sistema es **abierto a extensión** (Open/Closed Principle a nivel de arquitectura). Añadir capacidad no requiere modificar lo que ya funciona.

**English:**
This is the real power of event-driven decoupling: the system is **open for extension** (Open/Closed Principle at the architecture level). Adding capability does not require modifying what already works.

> **Regla de Pogrebinsky:** "If adding a new feature requires changing an existing service, your architecture is not event-driven enough. The goal is: publish an event once, and let the world react."
