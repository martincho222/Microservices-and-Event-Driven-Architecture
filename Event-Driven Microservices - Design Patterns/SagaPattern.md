# Saga Pattern

---

## 1. Problem Statement

**Español:**
En una arquitectura de microservicios, cada servicio tiene su propia base de datos. Esto es fundamental para el desacoplamiento — pero introduce un problema crítico: **¿cómo se mantiene la consistencia de datos cuando una operación de negocio involucra múltiples servicios?** En una aplicación monolítica, una transacción que abarca múltiples tablas es trivial gracias a ACID. En microservicios, esa misma operación cruza múltiples servicios independientes con bases de datos separadas, y no existe un gestor de transacciones distribuidas que los coordine de forma nativa.

**English:**
In a microservices architecture, each service owns its own database. This is fundamental for decoupling — but it introduces a critical problem: **how do you maintain data consistency when a business operation spans multiple services?** In a monolithic application, a transaction spanning multiple tables is trivial thanks to ACID. In microservices, that same operation crosses multiple independent services with separate databases, and there is no native distributed transaction manager to coordinate them.

### El problema en ShopFast

```
┌─────────────────────────────────────────────────────────────────┐
│         FLUJO DE UNA ORDEN EN SHOPFAST — SIN SAGA               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cliente hace checkout de un carrito de $250                    │
│                                                                 │
│  Paso 1: Orders Service     → crea la orden       ✔ (DB Orders) │
│  Paso 2: Payments Service   → cobra la tarjeta    ✔ (DB Payments)│
│  Paso 3: Inventory Service  → reserva el stock    ✔ (DB Inventory)│
│  Paso 4: Shipping Service   → crea el envío       ❌ FALLA       │
│                                                                 │
│  Estado del sistema después del fallo:                          │
│  ─────────────────────────────────────────────────             │
│  Orders DB:    orden CREADA       ← inconsistente              │
│  Payments DB:  cobro REALIZADO    ← cliente pagó               │
│  Inventory DB: stock RESERVADO    ← producto bloqueado         │
│  Shipping DB:  envío NO creado    ← fallo                      │
│                                                                 │
│  El cliente pagó pero no tiene envío.                           │
│  El stock está bloqueado sin una orden válida.                  │
│  → INCONSISTENCIA DISTRIBUIDA ⚠️                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Español:**
La solución clásica en bases de datos relacionales sería usar una **transacción distribuida con 2PC (Two-Phase Commit)**. El problema es que 2PC en microservicios introduce acoplamiento fuerte entre servicios, bloqueos largos de recursos, y un único punto de fallo coordinador. En sistemas de alta disponibilidad con decenas de servicios, 2PC es impracticable.

**English:**
The classic solution in relational databases would be to use a **distributed transaction with 2PC (Two-Phase Commit)**. The problem is that 2PC in microservices introduces tight coupling between services, long resource locks, and a single coordinator point of failure. In high-availability systems with dozens of services, 2PC is impractical.

```
┌──────────────────────────────────────────────────────────────────┐
│         POR QUÉ 2PC NO FUNCIONA EN MICROSERVICIOS                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✗ Acoplamiento síncrono: todos los servicios deben estar        │
│    disponibles al mismo tiempo para que la transacción proceda   │
│                                                                  │
│  ✗ Bloqueos: cada servicio mantiene un lock sobre sus recursos   │
│    hasta recibir la fase 2 del coordinador                       │
│                                                                  │
│  ✗ Single point of failure: si el coordinador cae en mitad       │
│    de la transacción, todos los participantes quedan bloqueados  │
│                                                                  │
│  ✗ No escala: con 10+ servicios, la ventana de bloqueo se vuelve │
│    demasiado larga para un sistema de alta disponibilidad        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "In microservices, you cannot have distributed ACID transactions without sacrificing availability and independence. The Saga pattern is the pragmatic trade-off: eventual consistency in exchange for availability, scalability, and autonomy."

---

## 2. Introduction to the Saga Pattern

**Español:**
El **Saga Pattern** es un patrón de diseño para gestionar transacciones distribuidas en microservicios. En lugar de una única transacción atómica, una Saga es una **secuencia de transacciones locales**, donde cada servicio realiza su operación y publica un evento o mensaje para desencadenar el paso siguiente. Si algún paso falla, la Saga ejecuta **transacciones compensatorias** para deshacer los pasos previos — el equivalente distribuido del ROLLBACK.

**English:**
The **Saga Pattern** is a design pattern for managing distributed transactions in microservices. Instead of a single atomic transaction, a Saga is a **sequence of local transactions**, where each service performs its operation and publishes an event or message to trigger the next step. If any step fails, the Saga executes **compensating transactions** to undo the previous steps — the distributed equivalent of a ROLLBACK.

### Anatomía de una Saga — ShopFast checkout

```
┌─────────────────────────────────────────────────────────────────┐
│            SAGA: FLUJO EXITOSO (happy path)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  T1: Orders      → crea orden          → publica order.created  │
│         ↓                                                       │
│  T2: Payments    → cobra tarjeta       → publica payment.done   │
│         ↓                                                       │
│  T3: Inventory   → reserva stock       → publica stock.reserved │
│         ↓                                                       │
│  T4: Shipping    → crea envío          → publica shipment.done  │
│                                                                 │
│  ✔ Saga completada — todos los servicios en estado consistente  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────┐
│         SAGA: FLUJO CON FALLO + COMPENSACIÓN                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  T1: Orders      → crea orden          → publica order.created  │
│         ↓                                                       │
│  T2: Payments    → cobra tarjeta       → publica payment.done   │
│         ↓                                                       │
│  T3: Inventory   → reserva stock       ❌ FALLA (sin stock)     │
│         ↓                                                       │
│  ── INICIO DE COMPENSACIÓN ────────────────────────────────     │
│                                                                 │
│  C2: Payments    → reembolsa tarjeta   ← cancela T2             │
│  C1: Orders      → cancela orden       ← cancela T1             │
│                                                                 │
│  ✔ Sistema vuelve a un estado consistente (diferente al inicio, │
│    pero sin datos corruptos ni pagos sin orden)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Transacciones compensatorias — concepto clave

**Español:**
Una **transacción compensatoria** no es un rollback técnico — es una nueva transacción de negocio que deshace semánticamente el efecto de la transacción original. El cobro ya ocurrió y está registrado en Stripe/banco; la compensación es un **reembolso**, que es una operación real y auditable. Esto es una distinción fundamental respecto al ROLLBACK de SQL.

**English:**
A **compensating transaction** is not a technical rollback — it is a new business transaction that semantically undoes the effect of the original transaction. The charge already happened and is recorded in Stripe/bank; the compensation is a **refund**, which is a real and auditable operation. This is a fundamental distinction from SQL ROLLBACK.

```
  Transacción original    →    Transacción compensatoria
  ──────────────────────────────────────────────────────
  Crear orden             →    Cancelar orden
  Cobrar tarjeta          →    Reembolsar tarjeta
  Reservar stock          →    Liberar reserva de stock
  Crear envío             →    Cancelar envío
  Acumular puntos loyalty →    Revertir puntos acumulados
```

**Español:**
El estado final de una Saga fallida no es idéntico al estado inicial — existen registros de la orden cancelada, del reembolso, de la reserva fallida. Esto es **consistencia eventual**, no consistencia estricta. El sistema alcanza un estado coherente para el negocio, pero puede tomar tiempo y deja un rastro de auditoría.

**English:**
The final state of a failed Saga is not identical to the initial state — there are records of the cancelled order, the refund, the failed reservation. This is **eventual consistency**, not strict consistency. The system reaches a coherent state for the business, but it may take time and leaves an audit trail.

> **Regla de Pogrebinsky:** "A compensating transaction is not an undo — it is a forward step that corrects the business state. Design your Sagas knowing that partial states are visible to the system during execution. This is the price of availability."

---

## 3. Two Ways to Implement the Saga Pattern

**Español:**
Existen dos estrategias para coordinar los pasos de una Saga: **Choreography** (coreografía) y **Orchestration** (orquestación). La diferencia fundamental está en quién conoce y controla el flujo completo de la transacción.

**English:**
There are two strategies for coordinating the steps of a Saga: **Choreography** and **Orchestration**. The fundamental difference lies in who knows and controls the complete transaction flow.

---

### Implementación 1 — Choreography (Event-Driven Architecture)

**Español:**
En la coreografía, **no existe un coordinador central**. Cada servicio escucha eventos del broker, realiza su transacción local, y publica un nuevo evento que desencadena el siguiente paso. El flujo emerge del comportamiento colectivo de los servicios — como bailarines que siguen la música sin un director.

**English:**
In choreography, **there is no central coordinator**. Each service listens for events from the broker, performs its local transaction, and publishes a new event that triggers the next step. The flow emerges from the collective behavior of the services — like dancers following the music without a conductor.

```
┌─────────────────────────────────────────────────────────────────┐
│              SAGA CHOREOGRAPHY — ShopFast Checkout              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cliente                                                        │
│     │                                                           │
│     │  POST /orders                                             │
│     ▼                                                           │
│  Orders ──── order.created ────────────────────▶ [Broker]       │
│                                                      │          │
│                              Payments ◀─────────────┘          │
│                              (escucha order.created)            │
│                                  │                             │
│                                  │ cobra tarjeta               │
│                                  │                             │
│                              payment.processed ──▶ [Broker]    │
│                                                      │          │
│                         Inventory ◀─────────────────┘          │
│                         (escucha payment.processed)             │
│                              │                                  │
│                              │ reserva stock                   │
│                              │                                  │
│                         stock.reserved ──────────▶ [Broker]    │
│                                                      │          │
│                    Shipping ◀────────────────────────┘          │
│                    (escucha stock.reserved)                     │
│                         │                                       │
│                         │ crea envío                           │
│                         │                                       │
│                    shipment.created ──────────────▶ [Broker]   │
│                                                                 │
│  ── FALLO: Inventory no tiene stock ──────────────────────      │
│                                                                 │
│  Inventory publica: stock.reservation.failed                    │
│  Payments escucha stock.reservation.failed → reembolsa → publi- │
│    ca payment.refunded                                          │
│  Orders escucha payment.refunded → cancela orden               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Implementación 2 — Orchestration (Commands)

**Español:**
En la orquestación, existe un componente central llamado **Saga Orchestrator** (o Saga Manager) que coordina todos los pasos. El orquestador conoce el flujo completo: llama a cada servicio en orden, espera la respuesta, y decide si avanzar o desencadenar la compensación. Los servicios participantes no saben que forman parte de una Saga — solo responden a comandos.

**English:**
In orchestration, there is a central component called the **Saga Orchestrator** (or Saga Manager) that coordinates all steps. The orchestrator knows the complete flow: it calls each service in order, waits for the response, and decides whether to proceed or trigger compensation. The participating services do not know they are part of a Saga — they only respond to commands.

```
┌─────────────────────────────────────────────────────────────────┐
│            SAGA ORCHESTRATION — ShopFast Checkout               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌─────────────────────┐                      │
│                    │  Saga Orchestrator  │                      │
│                    │  (Order Saga)       │                      │
│                    └──────────┬──────────┘                      │
│                               │                                 │
│          ┌────────────────────┼────────────────────┐            │
│          │                    │                    │            │
│          ▼                    ▼                    ▼            │
│   ┌─────────────┐    ┌──────────────┐    ┌──────────────┐       │
│   │   Payments  │    │  Inventory   │    │   Shipping   │       │
│   │   Service   │    │   Service    │    │   Service    │       │
│   └─────────────┘    └──────────────┘    └──────────────┘       │
│                                                                 │
│  Flujo del orquestador:                                         │
│  1. Envía comando: charge(orderId, amount) → Payments           │
│  2. Recibe respuesta: payment_success / payment_failed          │
│  3. Si OK → envía comando: reserve(orderId, items) → Inventory  │
│  4. Recibe respuesta: reserved / out_of_stock                   │
│  5. Si OK → envía comando: createShipment(orderId) → Shipping   │
│  6. Si cualquier paso falla → ejecuta compensación en reversa   │
│                                                                 │
│  ── FALLO en Inventory ───────────────────────────────────      │
│  Orquestador recibe: out_of_stock                               │
│  Orquestador envía: refund(orderId) → Payments                  │
│  Orquestador envía: cancelOrder(orderId) → Orders               │
│  Orquestador marca la Saga como COMPENSADA                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Comparación Choreography vs Orchestration

```
┌──────────────────────────┬──────────────────────────┬──────────────────────────┐
│ Dimensión                │ Choreography             │ Orchestration            │
├──────────────────────────┼──────────────────────────┼──────────────────────────┤
│ Coordinador central      │ No                       │ Sí (Saga Orchestrator)   │
│ Conocimiento del flujo   │ Distribuido en eventos   │ Centralizado             │
│ Acoplamiento             │ Acoplamiento por eventos │ Servicios acopl. al orch.│
│ Trazabilidad del flujo   │ Difícil (flujo implícito)│ Fácil (flujo explícito)  │
│ Debugging                │ Complejo                 │ Simple                   │
│ Escalabilidad            │ Alta                     │ El orchestrator escala   │
│ Single point of failure  │ No                       │ Posible (el orchestrator)│
│ Lógica de negocio        │ Dispersa en los servicios│ Concentrada en el orch.  │
│ Cambios en el flujo      │ Tocar múltiples servicios│ Solo tocar el orchestrator│
│ Adecuado para            │ Flujos simples (2-3 pasos)│ Flujos complejos (4+ pasos)│
└──────────────────────────┴──────────────────────────┴──────────────────────────┘
```

---

## 4. Pros y Contras

### Choreography — Pros y Contras

```
┌─────────────────────────────────────────────────────────────────┐
│                CHOREOGRAPHY — PROS Y CONTRAS                    │
├────────────────────────────┬────────────────────────────────────┤
│  ✔ PROS                    │  ✗ CONTRAS                         │
├────────────────────────────┼────────────────────────────────────┤
│ Desacoplamiento real:      │ Flujo implícito: nadie tiene una   │
│ los servicios no se        │ vista completa de la transacción.  │
│ conocen directamente.      │ El flujo vive en los eventos.      │
│                            │                                    │
│ Sin single point of        │ Debugging difícil: rastrear una    │
│ failure: no hay            │ Saga fallida requiere correlacionar │
│ coordinador que pueda      │ eventos de múltiples topics y logs  │
│ caerse.                    │ de múltiples servicios.            │
│                            │                                    │
│ Fácil de escalar: cada     │ Dependencias circulares ocultas:   │
│ servicio escala de         │ A escucha a B, B escucha a C,      │
│ forma independiente.       │ C escucha a A → ciclos difíciles   │
│                            │ de detectar.                       │
│ Natural en EDA: los        │                                    │
│ servicios ya publican y    │ Testing complejo: probar el flujo  │
│ consumen eventos.          │ completo requiere levantar todos   │
│                            │ los servicios y el broker.         │
│ Menor latencia: no hay     │                                    │
│ un orchestrator intermedio │ Lógica de negocio dispersa:        │
│ que añada hops.            │ cambiar el flujo implica tocar      │
│                            │ múltiples servicios.               │
└────────────────────────────┴────────────────────────────────────┘
```

### Orchestration — Pros y Contras

```
┌─────────────────────────────────────────────────────────────────┐
│                ORCHESTRATION — PROS Y CONTRAS                   │
├────────────────────────────┬────────────────────────────────────┤
│  ✔ PROS                    │  ✗ CONTRAS                         │
├────────────────────────────┼────────────────────────────────────┤
│ Flujo explícito: la lógica │ Acoplamiento al orchestrator:      │
│ completa de la Saga está   │ los servicios deben conocer al     │
│ en un solo lugar, fácil de │ orchestrator o al menos responder  │
│ leer y entender.           │ a sus comandos.                    │
│                            │                                    │
│ Debugging simple: un solo  │ Posible single point of failure:   │
│ componente a inspeccionar  │ si el orchestrator cae en mitad    │
│ para entender el estado    │ de la Saga, el estado puede        │
│ de la transacción.         │ quedar inconsistente.              │
│                            │                                    │
│ Cambios de flujo           │ Riesgo de "God service":           │
│ centralizados: modificar   │ el orchestrator puede crecer y     │
│ la Saga solo requiere      │ acumular demasiada lógica de       │
│ cambiar el orchestrator.   │ negocio → acoplamiento implícito.  │
│                            │                                    │
│ Manejo de errores claro:   │ Latencia adicional: cada paso      │
│ el orchestrator decide     │ pasa por el orchestrator, que      │
│ explícitamente qué         │ añade un hop de red.               │
│ compensar y cuándo.        │                                    │
│                            │ Más infraestructura: requiere      │
│ Adecuado para flujos       │ implementar, desplegar y operar    │
│ complejos con muchas       │ el orchestrator (ej: Temporal,     │
│ condiciones y branches.    │ AWS Step Functions).               │
└────────────────────────────┴────────────────────────────────────┘
```

### Herramientas para Saga Orchestration

| Herramienta | Tipo | Característica principal |
|-------------|------|--------------------------|
| **Temporal** | Open source / cloud | Workflows durables con reintentos automáticos y estado persistente |
| **AWS Step Functions** | Managed (AWS) | Orquestación serverless, integración nativa con servicios AWS |
| **Apache Camel** | Open source | Motor de integración/routing con soporte de Saga |
| **Eventuate Tram** | Open source | Framework específico para Saga en microservicios Java |
| **MassTransit** | Open source (.NET) | Saga state machine para .NET con soporte de múltiples brokers |

### Resumen de decisión

```
┌─────────────────────────────────────────────────────────────────┐
│               ¿CHOREOGRAPHY U ORCHESTRATION?                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Usar CHOREOGRAPHY cuando:                                      │
│  → El flujo tiene 2 o 3 pasos simples y lineales                │
│  → Los servicios ya están orientados a eventos (EDA nativo)     │
│  → El equipo prioriza el desacoplamiento máximo                 │
│  → No se anticipa que el flujo cambie frecuentemente            │
│                                                                 │
│  Usar ORCHESTRATION cuando:                                     │
│  → El flujo tiene 4+ pasos o lógica condicional compleja        │
│  → La trazabilidad y el debugging son prioridad                 │
│  → El equipo necesita ver el estado de la Saga en tiempo real   │
│  → El flujo de negocio cambia frecuentemente                    │
│  → Se usa un motor de workflows maduro (Temporal, Step Func.)   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Choreography gives you decoupling; orchestration gives you clarity. For simple flows, choreography is elegant. For complex business processes with multiple failure modes, an explicit orchestrator is worth the added infrastructure cost — you will thank yourself when debugging a production incident at 3am."
