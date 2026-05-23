# Event Sourcing Pattern

---

## 1. Problem Statement

**Español:**
En los sistemas tradicionales, la base de datos almacena únicamente el **estado actual** de las entidades. Cuando una orden pasa de `PENDING` a `PAID` a `SHIPPED`, la fila en la tabla se sobreescribe en cada transición. El estado anterior se pierde. No hay registro de cuándo ocurrió el cambio, quién lo provocó, ni qué valor tenía el campo antes. El sistema sabe **qué es** el mundo ahora, pero no **cómo llegó** a ser así.

**English:**
In traditional systems, the database stores only the **current state** of entities. When an order transitions from `PENDING` to `PAID` to `SHIPPED`, the row in the table is overwritten at each transition. The previous state is lost. There is no record of when the change occurred, who triggered it, or what the field value was before. The system knows **what** the world is now, but not **how it got there**.

### El problema en ShopFast

```
┌─────────────────────────────────────────────────────────────────┐
│         PERSISTENCIA TRADICIONAL — SOLO EL ESTADO ACTUAL        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tabla: orders                                                  │
│                                                                 │
│  order_id │ user_id │ total  │ status    │ updated_at           │
│  ─────────┼─────────┼────────┼───────────┼──────────────────── │
│  421      │ usr-88  │ 250.00 │ SHIPPED   │ 2024-11-29 14:32     │
│                                                                 │
│  Preguntas que el sistema NO puede responder:                   │
│                                                                 │
│  ✗ ¿Cuándo cambió el status de PENDING a PAID?                  │
│  ✗ ¿La orden fue cancelada y re-activada en algún momento?      │
│  ✗ ¿Cuál era el total original antes de aplicar el cupón?       │
│  ✗ ¿Qué servicio o usuario provocó el cambio a SHIPPED?         │
│  ✗ ¿La dirección de envío fue modificada después de confirmar?  │
│  ✗ ¿Cuántos reintentos de cobro hubo antes del exitoso?         │
│                                                                 │
│  Toda esta información se PERDIÓ en cada UPDATE.                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Español:**
Este problema se manifiesta en tres áreas críticas del negocio: **auditoría** (regulaciones financieras y legales exigen trazabilidad completa), **debugging** (reproducir un bug de producción es imposible sin saber la secuencia de eventos que lo causó), y **negocio** (el análisis del comportamiento de usuarios y órdenes requiere datos históricos, no solo el estado final).

**English:**
This problem manifests in three critical business areas: **auditing** (financial and legal regulations demand complete traceability), **debugging** (reproducing a production bug is impossible without knowing the sequence of events that caused it), and **business analysis** (understanding user and order behavior requires historical data, not just the final state).

```
┌─────────────────────────────────────────────────────────────────┐
│  CONSECUENCIAS DEL MODELO TRADICIONAL                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ Sin trazabilidad de auditoría                                │
│    → Imposible cumplir regulaciones financieras (PCI DSS,       │
│      SOX) que exigen registro inmutable de cada transacción     │
│                                                                 │
│  ✗ Sin capacidad de replay                                      │
│    → Si un consumer procesó mal los datos, no se puede          │
│      "volver atrás" y reprocesar desde el estado correcto       │
│                                                                 │
│  ✗ Sin temporal queries                                         │
│    → "¿Cuál era el saldo del cliente el 15 de octubre a las     │
│       10:00am?" es imposible de responder                       │
│                                                                 │
│  ✗ Debugging limitado                                           │
│    → Un bug que corrompió datos en producción no puede          │
│      reproducirse porque la secuencia de cambios se perdió      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "A system that only stores current state is optimized for answering 'what is the world now?' but blind to 'how did the world get here?' For any domain where history matters — financial, legal, operational — this is not a trade-off, it's a liability."

---

## 2. Introduction to Event Sourcing

**Español:**
**Event Sourcing** es un patrón de persistencia en el que el estado de una entidad no se almacena directamente, sino que se **deriva** de una secuencia de eventos inmutables almacenados en un **Event Store** (almacén de eventos). En lugar de hacer UPDATE sobre una fila, el sistema hace APPEND de un nuevo evento al log. El estado actual siempre puede reconstruirse reproduciendo (replaying) todos los eventos en orden.

**English:**
**Event Sourcing** is a persistence pattern in which the state of an entity is not stored directly, but is **derived** from an immutable sequence of events stored in an **Event Store**. Instead of performing an UPDATE on a row, the system APPENDs a new event to the log. The current state can always be reconstructed by replaying all events in order.

### El Event Store — estructura fundamental

```
┌─────────────────────────────────────────────────────────────────┐
│            EVENT STORE — SHOPFAST ORDER #421                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  seq │ eventType              │ timestamp           │ payload   │
│  ────┼────────────────────────┼─────────────────────┼─────────  │
│  1   │ OrderCreated           │ 2024-11-29 10:00:00 │ {         │
│      │                        │                     │   userId, │
│      │                        │                     │   items,  │
│      │                        │                     │   total:  │
│      │                        │                     │   300.00 }│
│  ────┼────────────────────────┼─────────────────────┼─────────  │
│  2   │ CouponApplied          │ 2024-11-29 10:00:05 │ {         │
│      │                        │                     │   coupon: │
│      │                        │                     │   "BF20", │
│      │                        │                     │   discount│
│      │                        │                     │   :50.00 }│
│  ────┼────────────────────────┼─────────────────────┼─────────  │
│  3   │ PaymentAttempted       │ 2024-11-29 10:00:10 │ { attempt │
│      │                        │                     │   :1,     │
│      │                        │                     │   result: │
│      │                        │                     │   FAILED }│
│  ────┼────────────────────────┼─────────────────────┼─────────  │
│  4   │ PaymentAttempted       │ 2024-11-29 10:00:40 │ { attempt │
│      │                        │                     │   :2,     │
│      │                        │                     │   result: │
│      │                        │                     │   SUCCESS}│
│  ────┼────────────────────────┼─────────────────────┼─────────  │
│  5   │ InventoryReserved      │ 2024-11-29 10:00:42 │ { items,  │
│      │                        │                     │   warehse }│
│  ────┼────────────────────────┼─────────────────────┼─────────  │
│  6   │ ShipmentCreated        │ 2024-11-29 11:15:00 │ { carrier,│
│      │                        │                     │   tracking}│
│  ────┼────────────────────────┼─────────────────────┼─────────  │
│  7   │ OrderShipped           │ 2024-11-29 14:32:00 │ { eta }   │
│                                                                 │
│  Estado actual = replay de eventos 1 → 7:                       │
│  { status: SHIPPED, total: 250.00, attempts: 2, ... }           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Reconstrucción del estado — cómo funciona el replay

```
  Estado inicial del aggregate Order:
  { orderId: null, status: null, total: 0, items: [] }

  + Aplicar OrderCreated:
  { orderId: "421", status: PENDING, total: 300.00, items: [...] }

  + Aplicar CouponApplied:
  { orderId: "421", status: PENDING, total: 250.00, coupon: "BF20" }

  + Aplicar PaymentAttempted (FAILED):
  { ..., paymentAttempts: 1, lastPaymentStatus: FAILED }

  + Aplicar PaymentAttempted (SUCCESS):
  { ..., status: PAID, paymentAttempts: 2, lastPaymentStatus: OK }

  + Aplicar InventoryReserved:
  { ..., status: PROCESSING, reservations: [...] }

  + Aplicar ShipmentCreated:
  { ..., trackingNumber: "UPS-9832", carrier: "UPS" }

  + Aplicar OrderShipped:
  { ..., status: SHIPPED, shippedAt: "2024-11-29T14:32:00Z" }

  Estado final reconstruido === estado en DB tradicional
  + TODO el historial completo disponible
```

### Propiedades fundamentales del Event Store

```
┌─────────────────────────────────────────────────────────────────┐
│              PROPIEDADES DEL EVENT STORE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. APPEND-ONLY: nunca se hace UPDATE ni DELETE de eventos.     │
│     Los eventos son hechos inmutables del pasado.               │
│                                                                 │
│  2. ORDERED: los eventos tienen un número de secuencia (seq)    │
│     que garantiza el orden de aplicación al reconstruir.        │
│                                                                 │
│  3. IMMUTABLE: una vez escrito, un evento no puede modificarse. │
│     Si se cometió un error, se corrige con un nuevo evento      │
│     compensatorio (ej: OrderCorrected, not UPDATE).             │
│                                                                 │
│  4. COMPLETE: el Event Store ES la fuente de verdad.            │
│     No hay otra tabla de "estado actual" — solo el log.         │
│     (aunque en la práctica se usan proyecciones para queries)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Event Sourcing + CQRS — Combinación Natural

**Español:**
Event Sourcing resuelve el problema de **cómo almacenar** el estado (log de eventos). Pero reconstruir el estado actual reproduciendo todos los eventos cada vez que se hace una query es ineficiente. Aquí es donde CQRS entra como complemento natural: el **Write Side** usa Event Sourcing (append al Event Store), y el **Read Side** usa proyecciones precalculadas (Read Models) que se actualizan consumiendo los eventos del store.

**English:**
Event Sourcing solves the problem of **how to store** state (event log). But reconstructing current state by replaying all events every time a query is made is inefficient. This is where CQRS enters as a natural complement: the **Write Side** uses Event Sourcing (append to Event Store), and the **Read Side** uses pre-calculated projections (Read Models) that update by consuming events from the store.

```
┌─────────────────────────────────────────────────────────────────┐
│              EVENT SOURCING + CQRS EN SHOPFAST                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WRITE SIDE:                                                    │
│    Cliente → Command (CreateOrder)                              │
│           → Orders Aggregate (valida reglas de negocio)         │
│           → Genera evento: OrderCreated                         │
│           → APPEND al Event Store (append-only)                 │
│                                                                 │
│                  Event Store                                    │
│                  (append-only)                                  │
│                       │                                         │
│         ┌─────────────┼─────────────┐                           │
│         ▼             ▼             ▼                            │
│                                                                 │
│  READ SIDE (proyecciones):                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Order Status │  │Order History │  │  Analytics   │           │
│  │  Projection  │  │  Projection  │  │  Projection  │           │
│  │  (Redis)     │  │  (MongoDB)   │  │ (ClickHouse) │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                 │
│  Cada proyección consume los eventos del store y mantiene       │
│  la vista optimizada para su caso de uso específico.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Snapshots — optimización para aggregates de larga vida

**Español:**
Si una orden tiene 500 eventos a lo largo de su vida, reconstruir el aggregate reproduciendo 500 eventos en cada operación es costoso. Los **Snapshots** son una optimización: periódicamente se serializa el estado actual del aggregate y se guarda como punto de partida. El replay solo necesita partir desde el último snapshot más los eventos posteriores a él.

**English:**
If an order has 500 events over its lifetime, reconstructing the aggregate by replaying 500 events on every operation is expensive. **Snapshots** are an optimization: the current state of the aggregate is periodically serialized and saved as a starting point. Replay only needs to start from the last snapshot plus events after it.

```
  Sin snapshot:
  ───────────────────────────────────────────────────────────
  Replay: evento 1 → 2 → 3 → ... → 500   (500 operaciones)

  Con snapshot cada 100 eventos:
  ───────────────────────────────────────────────────────────
  Cargar snapshot en seq=400  (estado precalculado)
  Replay: evento 401 → 402 → ... → 500   (solo 100 operaciones)

  Resultado: reconstrucción x5 más rápida para aggregates viejos.
```

---

## 4. Beneficios de Event Sourcing

```
┌─────────────────────────────────────────────────────────────────┐
│                  BENEFICIOS DE EVENT SOURCING                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✔ Auditoría completa e inmutable:                              │
│    Cada cambio de estado queda registrado con timestamp,        │
│    actor y payload. Cumplimiento regulatorio (PCI DSS, SOX,     │
│    GDPR) sin tablas de auditoría adicionales. El log ES         │
│    la auditoría.                                                │
│                                                                 │
│  ✔ Temporal queries (time travel):                              │
│    "¿Cuál era el estado de la orden #421 el 15 de nov a         │
│     las 10am?" → replay de eventos hasta esa fecha.             │
│    Imposible con persistencia tradicional.                      │
│                                                                 │
│  ✔ Replay para reconstruir Read Models:                         │
│    Si un Read Model tiene un bug o se introduce una nueva        │
│    proyección, se puede reconstruir desde cero reproduciendo    │
│    todos los eventos históricos. No se pierde información.      │
│                                                                 │
│  ✔ Debugging de producción:                                     │
│    Para reproducir un bug: tomar el Event Store del aggregate   │
│    afectado y replaying en un entorno de prueba. La secuencia   │
│    exacta de eventos que llevó al estado corrupto es visible.   │
│                                                                 │
│  ✔ Integración natural con EDA:                                 │
│    Los eventos del Event Store son los mismos que se publican   │
│    en el broker (Kafka). Event Sourcing y Event-Driven           │
│    Architecture usan el mismo modelo de eventos.                │
│                                                                 │
│  ✔ Eliminación del "impedance mismatch":                        │
│    En OOP, los objetos tienen comportamiento y estado. En SQL,  │
│    las tablas tienen filas. Event Sourcing almacena el          │
│    COMPORTAMIENTO (eventos) no el estado → no hay mismatch      │
│    entre el modelo de dominio y el modelo de persistencia.      │
│                                                                 │
│  ✔ Desacoplamiento entre Write y Read:                          │
│    El Event Store no sabe nada de cómo se consultan los datos.  │
│    Nuevas proyecciones se pueden añadir en cualquier momento    │
│    sin tocar el Write Side — replay desde el inicio.            │
│                                                                 │
│  ✔ Visualization (trazabilidad visual del ciclo de vida):       │
│    El Event Store es una línea de tiempo completa y legible     │
│    del aggregate. Se puede visualizar exactamente qué pasó,     │
│    en qué orden y cuándo. En ShopFast: la orden #421 muestra    │
│    OrderCreated → CouponApplied → PaymentFailed →               │
│    PaymentSuccess → Shipped — la historia completa de un        │
│    vistazo. Invaluable para soporte al cliente y operaciones.   │
│                                                                 │
│  ✔ Corrections (corrección sin pérdida de datos):               │
│    Los errores no se corrigen con UPDATE — se corrigen con un   │
│    nuevo evento compensatorio. Si una orden se procesó con      │
│    el precio incorrecto, se publica OrderPriceCorrected.        │
│    El historial preserva el error original Y la corrección,     │
│    con timestamp y actor. El audit trail nunca se borra.        │
│    Esto es fundamentalmente diferente a un UPDATE silencioso    │
│    que oculta que el error existió.                             │
│                                                                 │
│  ✔ High Write Performance (alto rendimiento de escritura):      │
│    Las escrituras al Event Store son siempre INSERT (append).   │
│    No hay UPDATE con locks de fila, no hay SELECT + UPDATE,     │
│    no hay índices secundarios que actualizar en la escritura.   │
│    Un append-only log puede alcanzar throughputs muy altos      │
│    con latencias muy bajas. En Kafka como Event Store:          │
│    millones de escrituras por segundo sin degradación.          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Contras de Event Sourcing

```
┌─────────────────────────────────────────────────────────────────┐
│                   CONTRAS DE EVENT SOURCING                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ Complejidad alta:                                            │
│    Event Sourcing requiere pensar en términos de eventos, no    │
│    de entidades. El modelo mental es diferente al CRUD clásico. │
│    Curva de aprendizaje significativa para el equipo.           │
│                                                                 │
│  ✗ Consultas directas son costosas sin proyecciones:            │
│    "Dame todas las órdenes con status SHIPPED del último mes"   │
│    → sin una proyección, requiere replay de todos los           │
│    aggregates → inviable en producción. CQRS es obligatorio.    │
│                                                                 │
│  ✗ Evolución del esquema de eventos (Event Versioning):          │
│    Si el payload de OrderCreated cambia en v2 (se añade un      │
│    campo requerido), los eventos antiguos en v1 siguen siendo   │
│    válidos y deben ser manejados. Requiere estrategias de        │
│    upcasting (transformar eventos viejos al nuevo formato).     │
│                                                                 │
│  ✗ Consistencia eventual en las proyecciones:                   │
│    El Read Model puede estar desactualizado hasta que consuma   │
│    los últimos eventos del store.                               │
│                                                                 │
│  ✗ El Event Store puede crecer enormemente:                     │
│    Aggregates de larga vida (cuentas bancarias con millones     │
│    de transacciones) generan logs muy grandes. Los Snapshots    │
│    mitigan el problema de rendimiento pero no el de storage.    │
│                                                                 │
│  ✗ Eliminar datos es estructuralmente difícil:                  │
│    El log es inmutable por diseño. Si un usuario ejerce el      │
│    "derecho al olvido" (GDPR), no se puede simplemente borrar   │
│    sus eventos. Requiere estrategias especiales (crypto-shredding│
│    — cifrar los eventos con una clave que luego se destruye).   │
│                                                                 │
│  ✗ Sobre-ingeniería para dominios sin historia:                 │
│    Si el negocio nunca necesitará auditoría ni replay, Event    │
│    Sourcing añade complejidad sin beneficio real.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Comparación: persistencia tradicional vs Event Sourcing

| Dimensión | Persistencia tradicional | Event Sourcing |
|-----------|-------------------------|----------------|
| Qué se almacena | Estado actual | Secuencia de eventos |
| Auditoría | Tabla adicional manual | Integrada — el log ES la auditoría |
| Temporal queries | Imposible | Nativo — replay hasta cualquier fecha |
| Replay de Read Models | Imposible | Sí — reconstrucción desde cero |
| Debugging de producción | Limitado | Reproducción exacta del estado |
| Evolución del schema | Migrations de columnas | Versionado de eventos (upcasting) |
| Complejidad | Baja | Alta |
| Eliminación de datos (GDPR) | DELETE estándar | Crypto-shredding requerido |
| Storage | Solo estado actual | Historial completo (crece con el tiempo) |
| Adecuado para | Dominios simples, CRUD | Dominios con historia, auditoría, regulados |

---

## 6. Cuándo usar Event Sourcing

```
┌─────────────────────────────────────────────────────────────────┐
│              ¿CUÁNDO APLICAR EVENT SOURCING?                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✔ USAR cuando:                                                 │
│  → El dominio tiene requerimientos de auditoría o compliance    │
│    (finanzas, salud, legal, e-commerce de alta escala)          │
│  → Se necesitan temporal queries ("estado en el pasado")        │
│  → Se anticipa la necesidad de nuevas proyecciones en el futuro │
│  → El debugging de bugs de estado en producción es crítico      │
│  → El equipo trabaja con DDD y modelos de dominio ricos          │
│  → Ya se usa EDA — los eventos del store y del broker coinciden │
│                                                                 │
│  ✗ NO USAR cuando:                                              │
│  → El servicio es CRUD puro sin lógica de dominio               │
│  → No hay requerimientos de auditoría ni historial              │
│  → El equipo no tiene experiencia con el patrón                 │
│    (alta curva de aprendizaje, riesgo de implementación incorrecta)│
│  → La eliminación de datos es frecuente (GDPR intensivo)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Decisión por servicio en ShopFast

| Servicio | ¿Event Sourcing? | Justificación |
|----------|-----------------|---------------|
| Orders | ✔ Sí | Ciclo de vida complejo, auditoría regulatoria, múltiples transiciones |
| Payments | ✔ Sí | Regulación financiera exige inmutabilidad y trazabilidad completa |
| Inventory | ⚠️ Condicional | Solo si se necesita historial de movimientos de stock |
| Users | ✗ No | CRUD básico, GDPR hace la eliminación problemática |
| Catalog | ✗ No | Datos de producto sin ciclo de vida complejo |
| Notifications | ✗ No | Sin requerimientos de historial ni auditoría |

> **Regla de Pogrebinsky:** "Event Sourcing is not a persistence strategy — it is a way of modeling your domain as a sequence of facts. Adopt it when your business needs the history of what happened, not just what is. The complexity cost is real: do not apply it universally. Apply it surgically, to the aggregates where history matters."