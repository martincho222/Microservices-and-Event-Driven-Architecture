# CQRS Pattern

---

## 1. Introduction to CQRS

**Español:**
**CQRS (Command Query Responsibility Segregation)** es un patrón de diseño que separa las operaciones de **escritura** (Commands) de las operaciones de **lectura** (Queries) en modelos distintos. En lugar de tener un único modelo que sirve tanto para modificar el estado como para consultarlo, CQRS propone dos modelos independientes: uno optimizado para recibir y procesar cambios, y otro optimizado para responder preguntas sobre el estado actual. Este patrón fue popularizado por Greg Young y es una extensión natural del principio CQS (Command Query Separation) de Bertrand Meyer.

**English:**
**CQRS (Command Query Responsibility Segregation)** is a design pattern that separates **write** operations (Commands) from **read** operations (Queries) into distinct models. Instead of having a single model that both modifies state and answers queries about it, CQRS proposes two independent models: one optimized to receive and process changes, and another optimized to answer questions about the current state. This pattern was popularized by Greg Young and is a natural extension of Bertrand Meyer's CQS (Command Query Separation) principle.

### El problema que resuelve

```
┌─────────────────────────────────────────────────────────────────┐
│         MODELO ÚNICO — EL PROBLEMA EN SHOPFAST                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Orders Service — modelo único sobre la misma DB:              │
│                                                                 │
│  ESCRITURAS:                                                    │
│    POST /orders         → INSERT en tabla orders (normalizada)  │
│    PATCH /orders/{id}   → UPDATE estado de la orden            │
│    POST /orders/cancel  → UPDATE + INSERT en order_events      │
│                                                                 │
│  LECTURAS (sobre el mismo modelo normalizado):                  │
│    GET /orders/{id}                → JOIN orders + items        │
│    GET /orders/history/{userId}    → JOIN orders + items        │
│                                      + payments + shipments     │
│                                      + GROUP BY + ORDER BY      │
│    GET /dashboard/admin            → agregaciones de 5 tablas   │
│                                      + 12 JOINs + ORDER BY      │
│                                                                 │
│  Problema:                                                      │
│  → El modelo normalizado es ideal para escritura (no repite     │
│    datos, integridad referencial) pero costoso para lectura.   │
│  → Las consultas complejas compiten en la misma DB con las      │
│    escrituras → contención de locks → latencia en ambas.       │
│  → Escalar la DB para soportar lecturas masivas afecta también  │
│    a las escrituras, y viceversa.                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### La solución CQRS

```
┌─────────────────────────────────────────────────────────────────┐
│                  CQRS — DOS MODELOS SEPARADOS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        ┌──────────────┐                         │
│   Cliente              │   API Layer  │                         │
│      │                 └──────┬───────┘                         │
│      │                        │                                 │
│      │         ┌──────────────┴──────────────┐                  │
│      │         │                             │                  │
│      │    COMMAND (escribe)            QUERY (lee)              │
│      │         │                             │                  │
│      │         ▼                             ▼                  │
│      │  ┌─────────────┐             ┌─────────────┐             │
│      │  │  Write Model│             │  Read Model │             │
│      │  │  (Commands) │             │  (Queries)  │             │
│      │  └──────┬──────┘             └──────┬──────┘             │
│      │         │                           │                    │
│      │         ▼                           ▼                    │
│      │  ┌─────────────┐             ┌─────────────┐             │
│      │  │  Write DB   │             │  Read DB    │             │
│      │  │ (normalizada│  ──sync──▶  │(denormalized│             │
│      │  │  ACID)      │  via events │ projection) │             │
│      │  └─────────────┘             └─────────────┘             │
│                                                                 │
│  Command: "crear orden", "cancelar orden", "actualizar precio"  │
│  Query:   "dame el historial", "dame el dashboard", "buscar"    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Español:**
La clave es que el **Write Model** está optimizado para integridad y consistencia — base de datos normalizada, validaciones, reglas de negocio. El **Read Model** está optimizado para velocidad de consulta — denormalizado, precalculado, con la forma exacta que necesita la UI. La sincronización entre los dos modelos ocurre de forma asíncrona, típicamente a través de eventos.

**English:**
The key is that the **Write Model** is optimized for integrity and consistency — normalized database, validations, business rules. The **Read Model** is optimized for query speed — denormalized, pre-calculated, shaped exactly as the UI needs it. Synchronization between the two models happens asynchronously, typically through events.

### Commands vs Queries — definición

```
  COMMAND (escritura):
  ────────────────────────────────────────────────────────────
  → Intención de cambiar el estado del sistema
  → Puede ser rechazado (validación fallida, regla de negocio)
  → No devuelve datos (idealmente solo éxito/fallo + eventId)
  → Ejemplos: CreateOrder, CancelOrder, UpdateProductPrice,
              RegisterUser, ProcessPayment

  QUERY (lectura):
  ────────────────────────────────────────────────────────────
  → Pregunta sobre el estado actual del sistema
  → NUNCA cambia el estado (idempotente por definición)
  → Devuelve datos optimizados para el caso de uso específico
  → Ejemplos: GetOrderHistory, GetProductDetails,
              GetAdminDashboard, SearchProducts
```

> **Regla de Pogrebinsky:** "A function that answers a question should not change the state of the system. A function that changes the state should not answer questions. When you mix both in the same model at scale, you optimize for neither."

---

## 2. CQRS Use Cases for Microservices Architecture

**Español:**
CQRS no es un patrón universal que deba aplicarse a todos los servicios. Tiene un costo real en complejidad operacional y consistencia eventual. Se justifica cuando el sistema presenta características específicas que hacen que un modelo único sea un cuello de botella.

**English:**
CQRS is not a universal pattern to be applied to every service. It has a real cost in operational complexity and eventual consistency. It is justified when the system presents specific characteristics that make a single model a bottleneck.

### Caso 1 — Asimetría extrema entre lecturas y escrituras

```
┌─────────────────────────────────────────────────────────────────┐
│  ShopFast — Catalog Service                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ESCRITURAS:   ~500 actualizaciones de precio/día               │
│                ~200 nuevos productos/día                        │
│                                                                 │
│  LECTURAS:     ~2,000,000 consultas de producto/día             │
│                (páginas de producto, búsquedas, comparadores)   │
│                                                                 │
│  Ratio lectura/escritura: 2000:1                                │
│                                                                 │
│  Sin CQRS: escalar la DB de escritura solo para soportar        │
│  lecturas es costoso e ineficiente.                             │
│                                                                 │
│  Con CQRS:                                                      │
│  → Write DB: PostgreSQL normalizado (500 writes/day, pequeño)   │
│  → Read DB:  Elasticsearch (2M reads/day, escala horizontal)    │
│  → Cada escritura publica product.updated → Read DB se actualiza│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Caso 2 — Múltiples representaciones del mismo dato

```
  ShopFast — Order data necesaria en distintos contextos:

  Vista del cliente (mobile):
  → orderId, status, items resumidos, estimated delivery
  → Formato: JSON compacto, < 1KB

  Vista del operador logístico:
  → orderId, items detallados, direcciones, instrucciones especiales
  → Formato: JSON extendido con todos los campos de shipping

  Dashboard de analytics:
  → Total de órdenes, revenue por período, top products
  → Formato: agregaciones pre-calculadas

  Sin CQRS: 3 queries complejas sobre el mismo modelo normalizado.
  Con CQRS: 3 Read Models distintos, cada uno con la forma
            exacta que necesita su consumer. 0 JOINs en lectura.
```

### Caso 3 — Necesidad de escalar lectura y escritura de forma independiente

```
┌─────────────────────────────────────────────────────────────────┐
│  Black Friday — ShopFast                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Lecturas: x50 del tráfico normal (browsing masivo)             │
│  Escrituras: x8 del tráfico normal (checkout)                   │
│                                                                 │
│  Sin CQRS: escalar la DB única implica escalar ambas           │
│  operaciones juntas → costoso, posible contención de locks.    │
│                                                                 │
│  Con CQRS:                                                      │
│  → Read Model (Elasticsearch/Redis): escalar x50 → 20 réplicas  │
│  → Write Model (PostgreSQL): escalar x8 → 3 instancias          │
│  → Escala independiente según la demanda real de cada lado.     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Caso 4 — Queries cross-service en microservicios

```
  Problema sin CQRS:
  ──────────────────────────────────────────────────────────────
  "Dame el historial de compras del usuario con nombre,
   status de cada orden, y puntos de loyalty acumulados"

  → Orders Service: SELECT * FROM orders WHERE user_id = ?
  → Users Service: GET /users/{id}              (llamada HTTP)
  → Loyalty Service: GET /points/{userId}       (llamada HTTP)
  → Frontend ensambla: 3 llamadas síncronas, 3 puntos de fallo

  Con CQRS — Read Model dedicado:
  ──────────────────────────────────────────────────────────────
  Un "Order History Read Model" mantiene una vista denormalizada:
  {
    userId, userName, email,
    orders: [{ orderId, status, items, total, loyaltyPoints }]
  }
  → Actualizado por eventos: order.created, order.status.updated,
    user.updated, loyalty.points.credited
  → 1 sola query, 0 llamadas cross-service en tiempo de lectura
```

### Caso 5 — Sistemas con alta complejidad de dominio (DDD)

**Español:**
En sistemas donde el Write Model implementa lógica de dominio compleja (aggregates, domain events, invariantes de negocio), el Read Model puede ser una proyección simple sin lógica. Esto permite que el modelo de escritura sea complejo y correcto, mientras que el modelo de lectura es simple y rápido. CQRS es el complemento natural de Domain-Driven Design (DDD) y Event Sourcing.

**English:**
In systems where the Write Model implements complex domain logic (aggregates, domain events, business invariants), the Read Model can be a simple projection with no logic. This allows the write model to be complex and correct, while the read model is simple and fast. CQRS is the natural complement to Domain-Driven Design (DDD) and Event Sourcing.

### Cuándo NO usar CQRS

```
┌─────────────────────────────────────────────────────────────────┐
│              CQRS NO ES ADECUADO CUANDO:                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ El dominio es simple (CRUD básico sin lógica compleja)       │
│    → El overhead de mantener dos modelos supera el beneficio    │
│                                                                 │
│  ✗ La consistencia inmediata es un requisito estricto           │
│    → CQRS introduce consistencia eventual en el Read Model      │
│    → Si el usuario debe ver su cambio reflejado al instante,    │
│      CQRS añade complejidad para un beneficio dudoso            │
│                                                                 │
│  ✗ El equipo es pequeño y el sistema está en etapa temprana     │
│    → Empezar con modelo único y migrar a CQRS cuando haya       │
│      evidencia de cuello de botella real                        │
│                                                                 │
│  ✗ El ratio lectura/escritura es ~1:1 sin picos                 │
│    → No hay asimetría que justifique la separación              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. CQRS Real-Life Example

**Español:**
El ejemplo más claro de CQRS en ShopFast es la separación entre el **Catalog Service** (write model) y el **Search Service** (read model). El catálogo es la fuente de verdad para los datos de productos — normalizado, consistente, con reglas de negocio. El Search Service mantiene una proyección denormalizada en Elasticsearch, optimizada para búsquedas full-text, filtros y rankings.

**English:**
The clearest CQRS example in ShopFast is the separation between the **Catalog Service** (write model) and the **Search Service** (read model). The catalog is the source of truth for product data — normalized, consistent, with business rules. The Search Service maintains a denormalized projection in Elasticsearch, optimized for full-text search, filters, and rankings.

### ShopFast — Catalog Write Model + Search Read Model

```
┌─────────────────────────────────────────────────────────────────┐
│        CQRS EN SHOPFAST: CATALOG + SEARCH                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  WRITE SIDE — Catalog Service                                   │
│  ──────────────────────────────────────────────────────────     │
│  Admin actualiza precio de producto:                            │
│                                                                 │
│  PUT /products/SKU-123/price  { price: 29.99 }                  │
│           │                                                     │
│           ▼                                                     │
│  Catalog Service:                                               │
│    1. Valida reglas de negocio (precio > 0, no inferior al      │
│       costo, no más de 80% de descuento)                        │
│    2. UPDATE products SET price=29.99 WHERE sku='SKU-123'       │
│       (PostgreSQL normalizado)                                  │
│    3. Publica: product.price.updated                            │
│       { productId, sku, newPrice, oldPrice, timestamp }         │
│           │                                                     │
│           │ event                                               │
│           ▼                                                     │
│        [Kafka broker]                                           │
│           │                                                     │
│  READ SIDE — Search Service                                     │
│  ──────────────────────────────────────────────────────────     │
│  Search Service consume product.price.updated:                  │
│                                                                 │
│    1. Lee el documento actual de Elasticsearch                  │
│    2. Actualiza el campo price en la proyección                 │
│    3. PUT /products/SKU-123 en Elasticsearch                    │
│       {                                                         │
│         productId, sku, name, description,                      │
│         price: 29.99,       ← actualizado                       │
│         category, brand, tags, images,                          │
│         averageRating, reviewCount,                             │
│         inStock, stockLevel,                                    │
│         popularityScore                                         │
│       }                                                         │
│    → Documento denormalizado listo para búsqueda en < 5ms       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Flujo completo de lectura vs escritura

```
  ESCRITURA (Command):
  ─────────────────────────────────────────────────────────────
  Admin App  →  PUT /catalog/products/{id}/price
             →  Catalog Service (validación + Write DB)
             →  product.price.updated (evento)
             →  Search Service actualiza Elasticsearch
             →  [otros consumers: Pricing, Recommendations...]

  Tiempo total de escritura: ~50ms (DB write + event publish)
  El admin recibe 200 OK al terminar la escritura.
  El Read Model se actualiza en los siguientes ~100-500ms.


  LECTURA (Query):
  ─────────────────────────────────────────────────────────────
  Usuario busca "auriculares bluetooth"
  →  GET /search?q=auriculares+bluetooth&filters=...
  →  Search Service consulta Elasticsearch directamente
  →  Elasticsearch devuelve documentos pre-indexados
  →  0 JOINs, 0 llamadas a Catalog Service
  →  Respuesta en < 10ms con 10,000 usuarios concurrentes
```

### El lag de consistencia eventual

```
┌─────────────────────────────────────────────────────────────────┐
│              CONSISTENCIA EVENTUAL EN CQRS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  t=0ms    Admin actualiza precio SKU-123 a $29.99               │
│           Write DB: precio = $29.99   ✔                         │
│           Read DB:  precio = $34.99   (aún no actualizado)      │
│                                                                 │
│  t=50ms   product.price.updated publicado en Kafka              │
│                                                                 │
│  t=200ms  Search Service consume el evento                      │
│           Read DB:  precio = $29.99   ✔ (sincronizado)          │
│                                                                 │
│  Ventana de inconsistencia: ~200ms                              │
│  → Un usuario que busque entre t=0 y t=200ms verá $34.99        │
│  → Después de t=200ms todos ven $29.99                          │
│                                                                 │
│  ¿Es aceptable?                                                 │
│  → Para un e-commerce: sí. El precio se actualizará en          │
│    milisegundos, antes de que el usuario haga checkout.         │
│  → Para un sistema de trading en tiempo real: no.               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Beneficios y Contras

### Beneficios

```
┌─────────────────────────────────────────────────────────────────┐
│                    BENEFICIOS DE CQRS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✔ Escalabilidad independiente:                                 │
│    El Read Model puede escalar horizontalmente de forma         │
│    independiente al Write Model. En ShopFast, Elasticsearch     │
│    puede tener 20 réplicas de lectura mientras PostgreSQL        │
│    mantiene 1 primario + 2 réplicas de escritura.               │
│                                                                 │
│  ✔ Modelos optimizados para su propósito:                       │
│    Write Model: normalizado, ACID, reglas de negocio, integridad│
│    Read Model: denormalizado, sin JOINs, con la forma exacta    │
│    que necesita la UI → latencia de lectura mínima.             │
│                                                                 │
│  ✔ Tecnologías distintas para cada lado:                        │
│    Write: PostgreSQL (consistencia transaccional)               │
│    Read:  Elasticsearch (full-text search), Redis (caché),      │
│           MongoDB (documentos flexibles), ClickHouse (analytics)│
│    Cada lado usa la herramienta correcta para su caso de uso.   │
│                                                                 │
│  ✔ Eliminación de contención de recursos:                       │
│    Las consultas complejas de lectura ya no compiten con las     │
│    escrituras en la misma DB → menos locks, mejor rendimiento   │
│    en ambos lados.                                              │
│                                                                 │
│  ✔ Múltiples vistas del mismo dato sin duplicar lógica:         │
│    Un solo Write Model puede alimentar N Read Models distintos  │
│    (mobile view, admin view, analytics view) vía eventos.       │
│                                                                 │
│  ✔ Facilita la integración con Event Sourcing:                  │
│    El Write Model publica eventos → los Read Models son          │
│    proyecciones que se pueden reconstruir desde cero en         │
│    cualquier momento usando el historial de eventos.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Contras

```
┌─────────────────────────────────────────────────────────────────┐
│                     CONTRAS DE CQRS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ Consistencia eventual:                                       │
│    El Read Model puede estar desactualizado por milliseconds    │
│    o segundos. Para operaciones donde el usuario debe ver su    │
│    cambio reflejado inmediatamente, se requieren workarounds    │
│    (optimistic UI update, polling, etc.).                       │
│                                                                 │
│  ✗ Complejidad operacional:                                     │
│    Dos modelos = dos bases de datos = dos pipelines de          │
│    sincronización = el doble de infraestructura a operar,       │
│    monitorear y mantener.                                       │
│                                                                 │
│  ✗ Más código:                                                  │
│    Además del Write Model, hay que implementar los consumers    │
│    que mantienen actualizados los Read Models, con manejo de    │
│    errores, idempotencia y reintentos.                          │
│                                                                 │
│  ✗ Riesgo de inconsistencia si el pipeline falla:               │
│    Si el consumer que actualiza el Read Model cae o tiene       │
│    un bug, el Read Model queda desactualizado indefinidamente   │
│    hasta que se detecte y corrija.                              │
│                                                                 │
│  ✗ Sobre-ingeniería para dominios simples:                      │
│    Un CRUD básico no necesita CQRS. Aplicarlo en servicios      │
│    simples añade complejidad sin beneficio tangible.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Comparación: modelo único vs CQRS

| Dimensión | Modelo único | CQRS |
|-----------|-------------|------|
| Complejidad inicial | Baja | Alta |
| Escalabilidad de lectura | Limitada (escala con escritura) | Independiente |
| Scalabilidad de escritura | Limitada (escala con lectura) | Independiente |
| Latencia de lectura | Media (JOINs en DB normalizada) | Mínima (proyección denormalizada) |
| Consistencia | Inmediata | Eventual (ms a segundos) |
| Tecnologías | Una DB para todo | DB por propósito |
| Código a mantener | Poco | Más (consumers de sincronización) |
| Adecuado para | Dominios simples, equipos pequeños | Sistemas de alta escala, lectura intensiva |
| Combinable con Event Sourcing | Sí, pero poco beneficio | Sí — complemento natural |

### Tabla de decisión para ShopFast

| Servicio | Ratio R/W | ¿Aplicar CQRS? | Justificación |
|----------|-----------|----------------|---------------|
| Catalog + Search | 2000:1 | ✔ Sí | Alta asimetría, tecnologías distintas (PG + ES) |
| Orders | 20:1 | ✔ Sí | Múltiples vistas (cliente, ops, analytics) |
| Users | 10:1 | ⚠️ Condicional | Solo si el dashboard admin lo exige |
| Payments | 2:1 | ✗ No | Requiere consistencia inmediata (regulatorio) |
| Notifications | 1:1 | ✗ No | CRUD simple, sin asimetría |
| Inventory | 5:1 | ⚠️ Condicional | Solo si el stock check es cuello de botella |

> **Regla de Pogrebinsky:** "CQRS is not about having two databases — it is about acknowledging that the model you need to write data correctly is rarely the model you need to read it efficiently. Apply it where the asymmetry is real and measurable, not as a default architectural choice."