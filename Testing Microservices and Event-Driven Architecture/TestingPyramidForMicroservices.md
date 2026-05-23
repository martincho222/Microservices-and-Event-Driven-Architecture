# Testing Pyramid for Microservices - Introduction and Challenges

---

## 1. Recap of the Testing Pyramid for Monolithic Applications

**Español:**
La **Pirámide de Testing** es un modelo propuesto por Mike Cohn que describe la proporción ideal de cada tipo de test en una aplicación. La idea central es simple: los tests que son más rápidos, más baratos y más fáciles de mantener deben ser los más numerosos (base de la pirámide), y los tests que son más lentos, costosos y frágiles deben ser los menos (cima de la pirámide).

**English:**
The **Testing Pyramid** is a model proposed by Mike Cohn that describes the ideal proportion of each type of test in an application. The central idea is simple: tests that are faster, cheaper, and easier to maintain should be the most numerous (base of the pyramid), and tests that are slower, more expensive, and more fragile should be the fewest (top of the pyramid).

```
┌─────────────────────────────────────────────────────────────────┐
│           PIRÁMIDE DE TESTING — APLICACIÓN MONOLÍTICA           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                         /\                                      │
│                        /  \                                     │
│                       / E2E\          ← Pocos (1-5%)            │
│                      /──────\           Lentos, frágiles        │
│                     / Integ. \                                  │
│                    /──────────\       ← Medianos (15-25%)       │
│                   /Integration \        Moderados               │
│                  /──────────────\                               │
│                 /                \                              │
│                /   Unit Tests    \   ← Muchos (70-80%)          │
│               /──────────────────\    Rápidos, baratos          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Capa 1 — Unit Tests (Tests Unitarios)

**Español:**
Un test unitario prueba **una sola unidad de código** — una función, un método, una clase — de forma completamente aislada. No se conecta a bases de datos, no hace llamadas HTTP, no levanta servidores. Es el test más básico y más valioso: si falla, sabes exactamente dónde está el problema.

**English:**
A unit test tests **a single unit of code** — a function, a method, a class — in complete isolation. It does not connect to databases, does not make HTTP calls, does not start servers. It is the most basic and most valuable test: if it fails, you know exactly where the problem is.

```
  Ejemplo ShopFast — Orders Service:

  // Función a testear
  function calculateOrderTotal(items, couponDiscount) {
    const subtotal = items.reduce((sum, item) => sum + item.price * item.qty, 0)
    return subtotal - couponDiscount
  }

  // Unit Test
  test("calcula el total correctamente con descuento", () => {
    const items = [{ price: 100, qty: 2 }, { price: 50, qty: 1 }]
    const result = calculateOrderTotal(items, 30)
    expect(result).toBe(220)  // (100*2 + 50*1) - 30 = 220
  })

  → No necesita DB, no necesita red, no necesita broker.
  → Se ejecuta en < 1ms. Se pueden correr 10,000 en 10 segundos.
```

**Características:**
- Velocidad: < 1ms por test
- Aislamiento: total (sin dependencias externas)
- Cantidad recomendada: 70-80% de todos los tests
- Cuándo fallan: identifican bugs en lógica de negocio pura

### Capa 2 — Integration Tests (Tests de Integración)

**Español:**
Un test de integración prueba cómo **dos o más componentes** funcionan juntos — por ejemplo, un servicio con su base de datos, o una función con un cliente HTTP real. Son más lentos porque necesitan levantar infraestructura real o simulada (una DB en memoria, un servidor mock).

**English:**
An integration test verifies how **two or more components** work together — for example, a service with its database, or a function with a real HTTP client. They are slower because they need to start real or simulated infrastructure (an in-memory DB, a mock server).

```
  Ejemplo ShopFast — Orders Service con PostgreSQL:

  // Integration Test
  test("guarda la orden en la DB y la recupera correctamente", async () => {
    // Levanta PostgreSQL en Docker (testcontainer)
    const order = await OrderRepository.save({
      userId: "usr-88",
      items: [{ productId: "SKU-123", qty: 1, price: 250 }],
      total: 250
    })

    const retrieved = await OrderRepository.findById(order.id)
    expect(retrieved.total).toBe(250)
    expect(retrieved.status).toBe("PENDING")
  })

  → Necesita una DB real (aunque sea en Docker temporal).
  → Se ejecuta en ~500ms. Se corren cientos, no miles.
```

**Características:**
- Velocidad: 100ms – 2s por test
- Aislamiento: parcial (prueba la interacción con 1 dependencia)
- Cantidad recomendada: 15-25% de todos los tests
- Cuándo fallan: identifican bugs en la integración con infraestructura

### Capa 3 — End-to-End Tests / E2E (Tests de Extremo a Extremo)

**Español:**
Un test E2E prueba el sistema completo desde la perspectiva del usuario final — simula una acción real del usuario (hacer un pedido, pagar, recibir confirmación) y verifica que todo el flujo funciona correctamente de principio a fin. Son los tests más costosos porque necesitan el sistema entero funcionando.

**English:**
An E2E test verifies the complete system from the end user's perspective — it simulates a real user action (placing an order, paying, receiving confirmation) and checks that the entire flow works correctly end to end. They are the most expensive tests because they require the entire system to be running.

```
  Ejemplo ShopFast — flujo completo de compra:

  // E2E Test
  test("el usuario puede completar una compra", async () => {
    // Necesita: Orders, Payments, Inventory, Shipping, Notifications
    //           + Kafka broker + todas las DBs + la UI

    await browser.goto("/products/SKU-123")
    await browser.click("Agregar al carrito")
    await browser.click("Checkout")
    await browser.fillForm({ card: "4111...", address: "..." })
    await browser.click("Confirmar compra")
    await browser.waitFor(".order-confirmed")
    expect(await browser.getText(".order-status")).toBe("Pedido confirmado")
  })

  → Necesita TODO el sistema corriendo.
  → Se ejecuta en 30-120 segundos por test.
  → Son los más frágiles: fallan por razones ajenas al código.
```

**Características:**
- Velocidad: 30s – 2min por test
- Aislamiento: ninguno (prueba el sistema completo)
- Cantidad recomendada: 1-5% de todos los tests
- Cuándo fallan: identifican bugs en la integración del sistema completo, pero a veces es difícil saber exactamente dónde

> **Regla de Pogrebinsky:** "The pyramid shape is intentional. If your test suite looks like an ice cream cone — many E2E tests and few unit tests — you have a problem. Slow, fragile tests that are expensive to maintain will be skipped, disabled, or ignored. Fast, cheap, reliable tests will be run on every commit."

---

## 2. How to Apply the Testing Pyramid to Microservices Architecture

**Español:**
En una arquitectura de microservicios, la pirámide de testing sigue siendo válida, pero **se aplica a nivel de cada servicio individualmente** y necesita dos capas nuevas que no existen en el monolito: los **Component Tests** y los **Contract Tests**. Estas dos capas son la solución de Pogrebinsky al problema de testear servicios que se comunican entre sí sin tener que levantar todo el sistema.

**English:**
In a microservices architecture, the testing pyramid is still valid, but **it applies at the level of each individual service** and requires two new layers that do not exist in the monolith: **Component Tests** and **Contract Tests**. These two layers are Pogrebinsky's solution to the problem of testing services that communicate with each other without having to start the entire system.

### La pirámide de microservicios

```
┌─────────────────────────────────────────────────────────────────┐
│           PIRÁMIDE DE TESTING — MICROSERVICIOS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                         /\                                      │
│                        /  \                                     │
│                       / E2E\          ← Muy pocos (1-2%)        │
│                      /──────\           Solo happy paths críticos│
│                     / Integr.\                                  │
│                    /──────────\       ← Pocos (5-10%)           │
│                   / Component  \        Por servicio            │
│                  /──────────────\                               │
│                 /   Contract     \  ← Medianos (10-20%)         │
│                /──────────────────\   Entre servicios           │
│               /   Unit Tests       \ ← Muchos (70-80%)          │
│              /──────────────────────\ Por servicio              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Capa 1 — Unit Tests (igual que en el monolito, por servicio)

**Español:**
Cada microservicio tiene su propia suite de unit tests. La regla es la misma: testear la lógica de negocio interna del servicio de forma completamente aislada. En ShopFast, el Orders Service tiene unit tests que no saben de Payments ni de Inventory — solo testean la lógica de órdenes.

**English:**
Each microservice has its own unit test suite. The rule is the same: test the internal business logic of the service in complete isolation. In ShopFast, the Orders Service has unit tests that know nothing about Payments or Inventory — they only test order logic.

### Capa 2 — Contract Tests (nueva capa para microservicios)

**Español:**
Un **Contract Test** (test de contrato) verifica que dos servicios que se comunican entre sí cumplen un **contrato acordado** — sin necesitar que ambos estén corriendo al mismo tiempo. El contrato define: qué endpoints expone el proveedor, qué formato tienen las requests y responses, y qué campos son obligatorios.

Herramienta estándar: **Pact** (consumer-driven contract testing).

**English:**
A **Contract Test** verifies that two services that communicate with each other comply with an **agreed contract** — without needing both to be running at the same time. The contract defines: what endpoints the provider exposes, what format the requests and responses have, and what fields are required.

Standard tool: **Pact** (consumer-driven contract testing).

```
┌─────────────────────────────────────────────────────────────────┐
│           CONTRACT TEST — ORDERS → PAYMENTS EN SHOPFAST         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONSUMER (Orders Service) define el contrato:                  │
│  "Cuando llamo a POST /payments/charge con este payload,        │
│   espero recibir una respuesta con esta estructura"             │
│                                                                 │
│  Contrato definido por Orders:                                  │
│  Request:  POST /payments/charge                                │
│            { orderId: "421", amount: 250.00, currency: "USD" }  │
│  Response: { chargeId: String, status: "SUCCESS"|"FAILED" }     │
│                                                                 │
│  PROVIDER TEST (Payments Service):                              │
│  Payments corre su test contra el contrato publicado por Orders  │
│  y verifica que su implementación lo cumple.                    │
│                                                                 │
│  Si Payments cambia el campo "chargeId" a "charge_id"           │
│  → el Contract Test falla ANTES de llegar a producción.         │
│  → Orders y Payments se desacoplan sin riesgo de ruptura.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Capa 3 — Component Tests (nuevo en microservicios)

**Español:**
Un **Component Test** prueba **un microservicio completo** — con todas sus capas internas (HTTP handler, lógica de negocio, repositorio, DB) — pero con todas sus **dependencias externas simuladas** (otros servicios mockeados, broker simulado). El objetivo es verificar que el servicio se comporta correctamente de principio a fin para cada caso de uso, sin necesitar que el resto del sistema esté levantado.

**English:**
A **Component Test** tests **a complete microservice** — with all its internal layers (HTTP handler, business logic, repository, DB) — but with all its **external dependencies mocked** (other services mocked, broker simulated). The goal is to verify that the service behaves correctly end-to-end for each use case, without needing the rest of the system to be up.

```
┌─────────────────────────────────────────────────────────────────┐
│        COMPONENT TEST — ORDERS SERVICE EN AISLAMIENTO           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Lo que se levanta:                                             │
│  ✔ Orders Service (código real)                                 │
│  ✔ PostgreSQL en Docker (DB real, pero solo para este test)     │
│                                                                 │
│  Lo que se simula (mocks/stubs):                                │
│  ✗ Payments Service     → mock HTTP que devuelve { status: OK } │
│  ✗ Inventory Service    → mock HTTP que devuelve { reserved: true}│
│  ✗ Kafka broker         → broker en memoria (TestContainers)    │
│  ✗ Shipping Service     → mock HTTP                             │
│                                                                 │
│  El test:                                                       │
│  POST /orders { userId: "usr-88", items: [...] }                │
│  → El servicio procesa la request completa                      │
│  → Escribe en PostgreSQL (real)                                 │
│  → Llama al mock de Payments (no el real)                       │
│  → Publica evento en Kafka en memoria                           │
│  → Responde 201 Created con el orderId                          │
│                                                                 │
│  Verifica: comportamiento completo del servicio en aislamiento   │
│  Sin verificar: la integración real con otros servicios         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Capa 4 — Integration Tests (entre servicios reales)

**Español:**
En microservicios, los integration tests verifican la comunicación real entre **dos servicios específicos** que se levantan juntos en un entorno de test. No es el sistema completo, solo el par de servicios cuya integración se quiere verificar. Son más lentos y escasos, pero prueban la comunicación real en vez de mocks.

**English:**
In microservices, integration tests verify real communication between **two specific services** that start together in a test environment. It is not the full system, just the pair of services whose integration needs verification. They are slower and fewer, but test real communication instead of mocks.

### Capa 5 — E2E Tests (el sistema completo)

**Español:**
En microservicios, los E2E tests son aún más escasos que en el monolito porque levantar todo el sistema (20 servicios + brokers + DBs) es muy costoso. Se reservan únicamente para los **happy paths más críticos del negocio** — el flujo de checkout completo, el login, el pago. Fallan frecuentemente por razones de infraestructura, no de código, por lo que su número debe ser mínimo.

**English:**
In microservices, E2E tests are even fewer than in the monolith because starting the entire system (20 services + brokers + DBs) is very expensive. They are reserved only for the **most critical business happy paths** — the complete checkout flow, login, payment. They frequently fail for infrastructure reasons rather than code issues, so their count must be minimal.

---

## 3. Challenges of Testing Microservices and Event-Driven Architecture

**Español:**
Testear microservicios es significativamente más difícil que testear un monolito. Los desafíos no son solo técnicos — son conceptuales. La distribución del sistema, la comunicación asíncrona, y la consistencia eventual crean escenarios de test que no existen en una aplicación monolítica.

**English:**
Testing microservices is significantly harder than testing a monolith. The challenges are not only technical — they are conceptual. System distribution, asynchronous communication, and eventual consistency create test scenarios that do not exist in a monolithic application.

### Desafío 1 — Dependencias entre servicios

```
┌─────────────────────────────────────────────────────────────────┐
│  PROBLEMA: Orders Service depende de 4 servicios                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Para testear Orders de forma realista, necesito:               │
│  → Payments Service corriendo y respondiendo                    │
│  → Inventory Service corriendo y respondiendo                   │
│  → Shipping Service corriendo y respondiendo                    │
│  → Notifications Service corriendo y respondiendo               │
│  + Kafka broker                                                 │
│  + 5 bases de datos                                             │
│                                                                 │
│  Si Payments está en desarrollo → mis tests bloquean           │
│  Si Inventory tiene un bug → mis tests fallan (falso positivo)  │
│  Si Kafka está caído → todos mis tests fallan                   │
│                                                                 │
│  SOLUCIÓN: Contract Tests + Component Tests con mocks           │
│  → Cada servicio testea su comportamiento en aislamiento        │
│  → Los contratos garantizan compatibilidad sin co-dependencia   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Desafío 2 — Comunicación asíncrona (el más difícil)

**Español:**
En sistemas síncronos, el test es simple: llamo a una función → recibo una respuesta → verifico el resultado. En sistemas event-driven, el flujo es: publico un evento → espero un tiempo desconocido → el consumer lo procesa → el estado cambia. El test no sabe cuánto esperar ni cómo verificar que el evento fue procesado correctamente.

**English:**
In synchronous systems, the test is simple: call a function → receive a response → verify the result. In event-driven systems, the flow is: publish an event → wait an unknown amount of time → the consumer processes it → the state changes. The test does not know how long to wait or how to verify that the event was processed correctly.

```
┌─────────────────────────────────────────────────────────────────┐
│         PROBLEMA DE ASINCRONÍA EN TESTS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  // ❌ Test INCORRECTO — race condition                          │
│  test("inventory se reserva cuando se crea una orden", async () => {
│    await ordersService.createOrder(orderData)                   │
│    // El evento order.created se publicó en Kafka               │
│    // ¿Cuándo lo procesa Inventory? ¿En 10ms? ¿En 500ms?        │
│    const reservation = await inventoryDB.findReservation(orderId)│
│    expect(reservation).toBeDefined()  // ← FALLA intermitente   │
│  })                                                             │
│                                                                 │
│  // ✔ Test CORRECTO — polling con timeout o await event         │
│  test("inventory se reserva cuando se crea una orden", async () => {
│    await ordersService.createOrder(orderData)                   │
│    // Espera hasta que la condición sea verdadera o timeout      │
│    await waitUntil(                                             │
│      () => inventoryDB.findReservation(orderId),                │
│      { timeout: 5000, interval: 100 }                           │
│    )                                                            │
│    const reservation = await inventoryDB.findReservation(orderId)│
│    expect(reservation).toBeDefined()  // ← Confiable            │
│  })                                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Desafío 3 — Consistencia eventual difícil de testear

**Español:**
En EDA, el estado final del sistema es **eventual** — puede estar en un estado inconsistente durante un período de tiempo. Un test que verifica el estado inmediatamente después de una operación puede fallar aunque el sistema esté funcionando correctamente. Hay que testear el estado **después** de que se estabilice, con mecanismos de espera apropiados.

**English:**
In EDA, the final state of the system is **eventual** — it may be in an inconsistent state for a period of time. A test that checks state immediately after an operation may fail even though the system is working correctly. State must be checked **after** it stabilizes, with appropriate wait mechanisms.

### Desafío 4 — Gestión de datos de test entre servicios

```
┌─────────────────────────────────────────────────────────────────┐
│       PROBLEMA: DATOS DE TEST DISTRIBUIDOS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Para testear el checkout de ShopFast necesito:                 │
│                                                                 │
│  Orders DB:    usuario con id "usr-88" debe existir             │
│  Catalog DB:   producto "SKU-123" con stock disponible          │
│  Payments DB:  método de pago válido para "usr-88"              │
│  Inventory DB: unidades disponibles de "SKU-123"                │
│  Users DB:     usuario "usr-88" autenticado y activo            │
│                                                                 │
│  En un monolito: un único INSERT en una transacción.            │
│  En microservicios: 5 operaciones en 5 DBs distintas,           │
│  sin transacción global, con cleanup después del test.          │
│                                                                 │
│  SOLUCIÓN: Test Data Builders + fixtures por servicio           │
│  + limpieza automática post-test por servicio                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Desafío 5 — Entorno de test costoso de mantener

**Español:**
Levantar 20 microservicios + Kafka + 20 bases de datos para correr los tests es lento, costoso en recursos y propenso a fallos de infraestructura. Los tests fallan no porque el código sea incorrecto, sino porque un contenedor tardó demasiado en arrancar o la red del CI no está disponible. Estos son los **false negatives** más frustrantes en microservicios.

**English:**
Starting 20 microservices + Kafka + 20 databases to run tests is slow, resource-intensive, and prone to infrastructure failures. Tests fail not because the code is wrong, but because a container took too long to start or the CI network was unavailable. These are the most frustrating **false negatives** in microservices.

### Desafío 6 — Testing de orden de eventos y condiciones de carrera

```
  Escenario en ShopFast:
  ──────────────────────────────────────────────────────────────
  El usuario cancela la orden (order.cancelled)
  Y simultáneamente el sistema procesa el envío (shipment.created)
  → ¿Qué pasa si shipment.created llega ANTES que order.cancelled?

  En producción los eventos pueden llegar fuera de orden.
  El test en entorno controlado siempre los entrega en orden.
  → El test pasa, producción falla. ⚠️

  SOLUCIÓN: tests de orden incorrecto de eventos (chaos testing)
  + diseño de consumers idempotentes que manejen cualquier orden.
```

---

## 4. Beneficios y Contras de la Pirámide de Testing en Microservicios

### Beneficios de aplicar la pirámide correctamente

```
┌─────────────────────────────────────────────────────────────────┐
│         BENEFICIOS DE LA PIRÁMIDE EN MICROSERVICIOS             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✔ Feedback rápido en el ciclo de desarrollo:                   │
│    Los unit tests se ejecutan en segundos. Un developer sabe    │
│    en < 10 segundos si su cambio rompió algo, sin esperar       │
│    a un pipeline de CI de 30 minutos.                           │
│                                                                 │
│  ✔ Cada servicio puede deployarse independientemente:           │
│    Con Contract Tests, el equipo de Payments puede hacer        │
│    deploy sin pedir al equipo de Orders que corra sus E2E.      │
│    Los contratos garantizan compatibilidad.                     │
│                                                                 │
│  ✔ Detección temprana de breaking changes:                      │
│    Si Payments cambia el formato de su API, el Contract Test    │
│    falla en el pipeline de Payments — antes de llegar a staging │
│    y mucho antes de llegar a producción.                        │
│                                                                 │
│  ✔ Tests confiables y deterministas:                            │
│    Unit y Component Tests son deterministas — siempre dan el    │
│    mismo resultado para el mismo input. No hay flakiness por    │
│    infraestructura, red o timing.                               │
│                                                                 │
│  ✔ Documentación viva del comportamiento:                       │
│    Los tests describen exactamente qué hace cada servicio.      │
│    Un desarrollador nuevo puede leer los tests del Orders       │
│    Service y entender su comportamiento sin leer toda la        │
│    implementación.                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Contras y desventajas

```
┌─────────────────────────────────────────────────────────────────┐
│           CONTRAS DE TESTING EN MICROSERVICIOS                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ Mayor complejidad de setup:                                  │
│    Cada servicio necesita su propia configuración de tests,     │
│    sus propios mocks, sus propios TestContainers. La inversión  │
│    inicial es significativamente mayor que en un monolito.      │
│                                                                 │
│  ✗ Los mocks pueden mentir:                                     │
│    Un mock de Payments que siempre devuelve { status: OK }      │
│    no refleja el comportamiento real de Payments bajo carga,    │
│    con timeouts o con errores de red. Los Component Tests       │
│    con mocks dan confianza, pero no garantías absolutas.        │
│                                                                 │
│  ✗ Contract Tests requieren disciplina de equipo:               │
│    Los contratos deben mantenerse actualizados. Si un equipo    │
│    no actualiza el contrato al cambiar su API, el sistema       │
│    de contratos falla y da falsa confianza al resto.            │
│                                                                 │
│  ✗ Tests E2E siguen siendo necesarios y siguen siendo lentos:   │
│    Por más que reduzcamos los E2E, siempre habrá un conjunto    │
│    mínimo de flujos críticos que solo se pueden validar con     │
│    el sistema completo corriendo.                               │
│                                                                 │
│  ✗ Testing de EDA es inherentemente más complejo:               │
│    Los tests asíncronos son no-deterministas por naturaleza.    │
│    Requerimientos de await, polling y timeouts hacen los tests  │
│    más difíciles de escribir y de mantener que los síncronos.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Comparación: Testing en Monolito vs Microservicios

| Dimensión | Monolito | Microservicios |
|-----------|----------|----------------|
| Unit Tests | Igual | Igual (por servicio) |
| Integration Tests | Con la DB del monolito | Con la DB del servicio |
| Nuevas capas | No | Contract Tests + Component Tests |
| E2E Tests | 1 sistema que levantar | 20+ servicios + brokers + DBs |
| Gestión de datos de test | 1 DB, 1 transacción | N DBs, sin transacción global |
| Velocidad de la suite completa | Minutos | Decenas de minutos |
| Fallos de infraestructura | Raros | Más frecuentes |
| Confianza en mocks | Alta | Media (mocks pueden mentir) |
| Detectar breaking changes | Fácil (mismo proceso) | Requiere Contract Tests explícitos |

### Resumen — la regla de oro

```
┌─────────────────────────────────────────────────────────────────┐
│              REGLA DE ORO PARA MICROSERVICIOS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Por cada microservicio:                                        │
│                                                                 │
│  ✔ Muchos Unit Tests       → lógica de negocio interna          │
│  ✔ Contract Tests          → contratos con cada dependencia     │
│  ✔ Component Tests         → servicio completo con mocks        │
│  ✔ Pocos Integration Tests → integración real con 1 dependencia │
│  ✔ Muy pocos E2E Tests      → solo happy paths críticos         │
│                                                                 │
│  Anti-patterns a evitar:                                        │
│  ✗ Solo E2E tests (ice cream cone anti-pattern)                 │
│  ✗ Mocks sin Contract Tests (confianza falsa)                   │
│  ✗ Tests que duermen (Thread.sleep / setTimeout fijo)           │
│  ✗ Tests que dependen del orden de ejecución                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "In microservices, you cannot test your way to confidence with E2E tests alone — there are too many services, too many combinations, and too many failure modes. The solution is to push confidence down the pyramid: strong unit tests for business logic, contract tests for service boundaries, and component tests for each service in isolation. Reserve E2E tests for the handful of flows where business risk justifies the cost."