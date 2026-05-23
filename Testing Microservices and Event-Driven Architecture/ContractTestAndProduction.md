# Contract Tests and Production Testing

---

## 1. Integration Tests Using Lightweight Mocks

**Español:**
Cuando un microservicio necesita comunicarse con otros servicios para funcionar, testear esa comunicación plantea un dilema: ¿levanto los servicios reales o los simulo? Levantar los servicios reales hace el test lento, frágil y dependiente de que todos estén disponibles. La solución son los **lightweight mocks** — servidores HTTP simulados que se levantan en milisegundos dentro del proceso del test, responden a las mismas rutas que el servicio real, y se controlan completamente desde el código del test.

**English:**
When a microservice needs to communicate with other services to function, testing that communication poses a dilemma: do I start the real services or simulate them? Starting real services makes tests slow, fragile, and dependent on all of them being available. The solution is **lightweight mocks** — simulated HTTP servers that start in milliseconds inside the test process, respond to the same routes as the real service, and are fully controlled from test code.

### Herramientas principales

| Herramienta | Lenguaje | Descripción |
|-------------|----------|-------------|
| **WireMock** | Java / JVM | Mock HTTP server más popular del ecosistema Java |
| **MockServer** | Java / Node | Soporta HTTP y HTTPS, con verificación de llamadas |
| **Nock** | Node.js | Intercepta llamadas HTTP a nivel de módulo `http` |
| **responses** | Python | Intercepta llamadas de la librería `requests` |
| **MSW (Mock Service Worker)** | JS/TS | Mock en browser y Node, ideal para frontend + backend |

### Cómo funciona un lightweight mock

```
┌─────────────────────────────────────────────────────────────────┐
│        INTEGRATION TEST CON WIREMOCK — SHOPFAST                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Objetivo: testear que Orders Service llama correctamente a     │
│  Payments Service al crear una orden                            │
│                                                                 │
│  SIN lightweight mock:                                          │
│  Test → Orders Service → Payments Service (real) → DB Payments  │
│  → Necesita Payments corriendo + su DB + su config              │
│  → Si Payments está caído: test falla (falso negativo)          │
│                                                                 │
│  CON lightweight mock (WireMock):                               │
│  Test → Orders Service → WireMock (simula Payments) → responde  │
│  → WireMock arranca en < 100ms dentro del test                  │
│  → Siempre disponible, siempre responde igual                   │
│  → El test es determinista y rápido                             │
│                                                                 │
│  // Configuración del mock                                      │
│  wireMock.stubFor(                                              │
│    post(urlEqualTo("/payments/charge"))                         │
│      .withRequestBody(matchingJsonPath("$.orderId"))            │
│      .willReturn(aResponse()                                    │
│        .withStatus(200)                                         │
│        .withBody("""                                            │
│          { "chargeId": "ch-999", "status": "SUCCESS" }          │
│        """))                                                    │
│  )                                                              │
│                                                                 │
│  // El test                                                     │
│  POST /orders { userId: "usr-88", items: [...], total: 250 }    │
│  → Orders llama al WireMock en vez de al Payments real          │
│  → WireMock responde con el stub definido                       │
│  → Orders procesa la respuesta y crea la orden                  │
│  → Test verifica: orden creada + llamada a Payments fue hecha   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Verificación de llamadas — no solo simular, sino verificar

```
  // Verificar que Orders SÍ llamó a Payments exactamente 1 vez
  // con el payload correcto

  wireMock.verify(1, postRequestedFor(urlEqualTo("/payments/charge"))
    .withRequestBody(matchingJsonPath("$.orderId", equalTo("421")))
    .withRequestBody(matchingJsonPath("$.amount", equalTo("250.00")))
  )

  → Si Orders no llamó a Payments → test falla ✔ (detecta regresión)
  → Si Orders llamó con el amount incorrecto → test falla ✔
  → Si Orders llamó dos veces (bug) → test falla ✔
```

### Beneficios y desventajas de lightweight mocks

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Tests rápidos (< 1s)         │ El mock puede no reflejar el     │
│ Sin dependencias externas    │ comportamiento real del servicio  │
│ Deterministas — siempre      │ bajo carga, timeouts o errores   │
│ mismo resultado              │ de red reales.                   │
│                              │                                  │
│ Control total: simula        │ Si el servicio real cambia su    │
│ errores, timeouts, códigos   │ API y el mock no se actualiza    │
│ HTTP específicos             │ → confianza falsa (mock lie).    │
│                              │                                  │
│ Tests en paralelo sin        │ Requiere mantener los stubs      │
│ interferencia entre          │ sincronizados con el servicio    │
│ instancias                   │ real → carga de mantenimiento.   │
│                              │                                  │
│ Simula casos de error        │ No detecta problemas de          │
│ difíciles de reproducir      │ serialización/deserialización    │
│ con servicios reales         │ hasta producción.                │
└──────────────────────────────┴──────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Lightweight mocks give you speed and isolation, but they carry a hidden risk: a mock that lies about the real service's behavior is worse than no mock at all. The solution is to pair mocks with contract tests — the contracts keep the mocks honest."

---

## 2. Contract Tests for Synchronous Communication

**Español:**
Los **Contract Tests síncronos** (también llamados Consumer-Driven Contract Tests) resuelven el problema del mock desactualizado. Funcionan así: el **consumer** (el servicio que hace la llamada) define formalmente qué espera recibir del **provider** (el servicio que responde). Ese contrato se publica en un registro compartido. El **provider** corre sus tests contra ese contrato y verifica que su implementación lo cumple. Si el provider cambia su API de forma que rompe el contrato → el test falla automáticamente, antes de llegar a producción.

**English:**
**Synchronous Contract Tests** (also called Consumer-Driven Contract Tests) solve the stale mock problem. They work as follows: the **consumer** (the calling service) formally defines what it expects to receive from the **provider** (the responding service). That contract is published to a shared registry. The **provider** runs its tests against that contract and verifies that its implementation complies. If the provider changes its API in a way that breaks the contract → the test fails automatically, before reaching production.

### Herramienta estándar: Pact

```
┌─────────────────────────────────────────────────────────────────┐
│        PACT CONTRACT TEST — ORDERS (consumer) + PAYMENTS        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PASO 1 — Consumer (Orders Service) define el contrato:         │
│  ──────────────────────────────────────────────────────────     │
│                                                                 │
│  // Orders dice: "necesito que Payments cumpla esto"            │
│  const interaction = pact.addInteraction({                      │
│    state: "payment service is available",                       │
│    uponReceiving: "a request to charge an order",               │
│    withRequest: {                                               │
│      method: "POST",                                            │
│      path: "/payments/charge",                                  │
│      body: {                                                    │
│        orderId: like("421"),        // cualquier string          │
│        amount:  like(250.00),       // cualquier número         │
│        currency: "USD"              // exactamente "USD"        │
│      }                                                          │
│    },                                                           │
│    willRespondWith: {                                           │
│      status: 200,                                               │
│      body: {                                                    │
│        chargeId: like("ch-999"),    // cualquier string          │
│        status: term({ matcher:      // SUCCESS o FAILED         │
│          "SUCCESS|FAILED",                                      │
│          generate: "SUCCESS" })                                 │
│      }                                                          │
│    }                                                            │
│  })                                                             │
│                                                                 │
│  → Pact genera un archivo JSON: orders-payments.pact.json       │
│  → Se publica en el Pact Broker (registro central)              │
│                                                                 │
│  PASO 2 — Provider (Payments Service) verifica el contrato:     │
│  ──────────────────────────────────────────────────────────     │
│                                                                 │
│  // En el pipeline de Payments, antes de cada deploy:           │
│  pact.verifyProvider({                                          │
│    provider: "PaymentsService",                                 │
│    pactBrokerUrl: "https://pact-broker.shopfast.internal",      │
│    publishVerificationResults: true                             │
│  })                                                             │
│                                                                 │
│  → Pact descarga el contrato de Orders                          │
│  → Hace las requests reales al Payments Service levantado       │
│  → Verifica que las respuestas cumplen el contrato              │
│  → Si Payments cambió "chargeId" a "charge_id" → FALLA ✔        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### El flujo completo con Pact Broker

```
  Consumer (Orders)             Pact Broker          Provider (Payments)
       │                            │                       │
       │── genera contrato ────────▶│                       │
       │   orders-payments.pact     │                       │
       │                            │                       │
       │                            │◀─── descarga ─────────│
       │                            │     contrato          │
       │                            │                       │
       │                            │      corre tests      │
       │                            │      contra contrato  │
       │                            │                       │
       │                            │──── publica ─────────▶│
       │                            │     resultado         │
       │                            │     (pass/fail)       │
       │                            │                       │
       │◀── notifica: "can I deploy?"│                      │
       │    "Payments cumple tu      │                      │
       │     contrato: YES"          │                      │
```

### Beneficios y desventajas de Contract Tests síncronos

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Detecta breaking changes     │ Requiere infraestructura:        │
│ antes de producción, en el   │ Pact Broker para gestionar y     │
│ pipeline del provider.       │ versionar los contratos.         │
│                              │                                  │
│ Deploy independiente con     │ Curva de aprendizaje: el equipo  │
│ confianza: "can I deploy?"   │ debe entender Pact y el flujo    │
│ tiene una respuesta objetiva.│ consumer-driven.                 │
│                              │                                  │
│ Los mocks del consumer       │ Solo cubre el contrato acordado: │
│ siempre están sincronizados  │ no detecta bugs en la lógica     │
│ con la implementación real   │ interna del provider.            │
│ del provider.                │                                  │
│                              │                                  │
│ Documentación viva de las    │ Disciplina de equipo necesaria:  │
│ APIs internas — los          │ si el consumer no actualiza el   │
│ contratos son el spec.       │ contrato, la garantía se rompe.  │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 3. Contract Tests for Asynchronous Communication

**Español:**
En sistemas event-driven, los servicios no se llaman directamente — publican eventos en un broker y otros servicios los consumen. El contrato ya no es una request/response HTTP sino un **esquema de evento**: qué campos contiene, qué tipos tienen, cuáles son obligatorios. Un **Contract Test asíncrono** verifica que el producer publica eventos con el formato correcto, y que el consumer puede procesar eventos con ese formato.

**English:**
In event-driven systems, services do not call each other directly — they publish events to a broker and other services consume them. The contract is no longer an HTTP request/response but an **event schema**: what fields it contains, what types they are, which are required. An **asynchronous Contract Test** verifies that the producer publishes events with the correct format, and that the consumer can process events with that format.

### El problema que resuelve

```
┌─────────────────────────────────────────────────────────────────┐
│  SIN CONTRACT TESTS ASÍNCRONOS — SHOPFAST                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Orders publica: order.created                                  │
│  {                                                              │
│    "orderId": "421",                                            │
│    "userId": "usr-88",                                          │
│    "totalAmount": 250.00    ← campo publicado por Orders        │
│  }                                                              │
│                                                                 │
│  Payments consume order.created y espera:                       │
│  {                                                              │
│    "orderId": ...,                                              │
│    "userId": ...,                                               │
│    "amount": ...            ← Payments usa "amount", no        │
│  }                             "totalAmount"                    │
│                                                                 │
│  Resultado en producción:                                       │
│  → Payments recibe el evento → amount === undefined             │
│  → Payments no cobra la tarjeta                                 │
│  → La orden se crea pero nunca se paga. ⚠️                     │
│                                                                 │
│  Este bug NO se detecta con unit tests ni component tests.      │
│  Solo se detecta en producción o con contract tests.            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Pact para mensajería asíncrona

```
┌─────────────────────────────────────────────────────────────────┐
│     CONTRACT TEST ASÍNCRONO — ORDERS (producer) + PAYMENTS      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PASO 1 — Consumer (Payments) define qué evento espera:         │
│  ──────────────────────────────────────────────────────────     │
│                                                                 │
│  // Payments dice: "espero recibir order.created con esta forma" │
│  const messagePact = pact.addMessage({                          │
│    description: "an order.created event",                       │
│    content: {                                                   │
│      orderId:     like("421"),                                  │
│      userId:      like("usr-88"),                               │
│      totalAmount: like(250.00),   // Payments sabe que es       │
│      currency:    "USD"           // "totalAmount", no "amount" │
│    }                                                            │
│  })                                                             │
│                                                                 │
│  // Test del lado del consumer:                                 │
│  // Payments recibe el mensaje del contrato y verifica          │
│  // que su handler lo puede procesar sin errores                │
│  messagePact.verify(message => {                                │
│    paymentsHandler.processOrderCreated(message.content)         │
│    // si lanza excepción → test falla                           │
│  })                                                             │
│                                                                 │
│  PASO 2 — Producer (Orders) verifica que publica ese contrato:  │
│  ──────────────────────────────────────────────────────────     │
│                                                                 │
│  // En el pipeline de Orders:                                   │
│  // Orders crea una orden de test y verifica que el evento      │
│  // generado cumple el contrato publicado por Payments           │
│  const event = ordersService.createOrder(testOrderData)         │
│  expect(event).toMatchSchema(paymentsContract.content)          │
│  // si "totalAmount" falta o es de tipo incorrecto → FALLA ✔    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Schema Registry — alternativa para eventos de alto volumen

**Español:**
En sistemas con muchos eventos y muchos consumers (como ShopFast con Kafka), una alternativa al Pact para mensajería es el **Schema Registry** de Confluent. Cada evento publicado en Kafka debe cumplir un esquema registrado (Avro, Protobuf o JSON Schema). Si el producer intenta publicar un evento que no cumple el esquema → la publicación falla inmediatamente. Si el esquema cambia de forma incompatible → el registro rechaza el cambio.

**English:**
In systems with many events and many consumers (like ShopFast with Kafka), an alternative to Pact for messaging is Confluent's **Schema Registry**. Every event published to Kafka must comply with a registered schema (Avro, Protobuf, or JSON Schema). If the producer tries to publish an event that does not comply with the schema → the publish fails immediately. If the schema changes in an incompatible way → the registry rejects the change.

```
┌─────────────────────────────────────────────────────────────────┐
│              SCHEMA REGISTRY — FLUJO EN SHOPFAST                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Orders quiere publicar order.created v2                        │
│  (cambió "totalAmount" a "amount" — BREAKING CHANGE)            │
│                                                                 │
│  Orders → Schema Registry: "¿puedo registrar este nuevo schema?"│
│                                                                 │
│  Schema Registry evalúa compatibilidad (BACKWARD mode):         │
│  → ¿Los consumers existentes pueden leer el nuevo schema?       │
│  → "totalAmount" desapareció → consumers que lo usan romperán   │
│  → REJECTED ✔ — el producer no puede publicar                  │
│                                                                 │
│  Modos de compatibilidad:                                       │
│  BACKWARD:  consumers nuevos pueden leer eventos viejos         │
│  FORWARD:   consumers viejos pueden leer eventos nuevos         │
│  FULL:      ambos (más restrictivo, más seguro)                 │
│  NONE:      sin validación (no recomendado en producción)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Beneficios y desventajas de Contract Tests asíncronos

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Detecta incompatibilidades   │ Más complejo de configurar que   │
│ de schema en el pipeline,    │ contract tests síncronos por la  │
│ no en producción.            │ naturaleza asíncrona del sistema. │
│                              │                                  │
│ Con Schema Registry: la      │ Schema Registry requiere         │
│ protección es automática     │ infraestructura adicional y      │
│ en cada publicación real.    │ que todos los servicios estén    │
│                              │ integrados con él.               │
│ Permite evolucionar eventos  │                                  │
│ de forma controlada con      │ Los contratos cubren el formato  │
│ reglas de compatibilidad     │ del evento, no la semántica:     │
│ (BACKWARD, FORWARD, FULL).   │ el campo puede existir pero con  │
│                              │ un valor incorrecto.             │
│ Documentación del schema     │                                  │
│ de todos los eventos como    │ Gestión de versiones de esquemas │
│ artefacto versionado.        │ en equipos grandes es compleja.  │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 4. Production Testing: Blue/Green Deployment and Canary Testing

**Español:**
Incluso con todos los tests anteriores, ningún entorno de test replica perfectamente producción — el volumen de tráfico real, los datos reales, los patrones de uso reales. El **Production Testing** es la práctica de testear en producción de forma controlada y segura, minimizando el impacto de los errores a un subconjunto de usuarios mientras se valida que el nuevo código funciona correctamente con tráfico real.

**English:**
Even with all the tests above, no test environment perfectly replicates production — real traffic volume, real data, real usage patterns. **Production Testing** is the practice of testing in production in a controlled and safe manner, minimizing the impact of errors to a subset of users while validating that the new code works correctly with real traffic.

---

### Estrategia 1 — Blue/Green Deployment

**Español:**
**Blue/Green Deployment** es una estrategia de despliegue en la que se mantienen **dos entornos de producción idénticos** en todo momento: el entorno **Blue** (la versión actual, en producción) y el entorno **Green** (la nueva versión, en espera). El cambio de versión consiste en redirigir el tráfico del balanceador de carga de Blue a Green en un solo switch — sin downtime. Si algo falla, el rollback es instantáneo: volver a apuntar el tráfico a Blue.

**English:**
**Blue/Green Deployment** is a deployment strategy where **two identical production environments** are maintained at all times: the **Blue** environment (current version, live) and the **Green** environment (new version, on standby). The version switch consists of redirecting traffic from the load balancer from Blue to Green in a single switch — with no downtime. If something fails, rollback is instant: point traffic back to Blue.

```
┌─────────────────────────────────────────────────────────────────┐
│           BLUE/GREEN DEPLOYMENT — SHOPFAST                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ANTES DEL DEPLOY:                                              │
│                                                                 │
│  Internet ──▶ Load Balancer ──▶ [BLUE]  Orders Service v1.4     │
│                                         (100% del tráfico)      │
│                                [GREEN] Orders Service v1.5      │
│                                         (0% del tráfico)        │
│                                         (listo, testeado)       │
│                                                                 │
│  SWITCH (< 1 segundo):                                          │
│                                                                 │
│  Internet ──▶ Load Balancer ──▶ [BLUE]  Orders Service v1.4     │
│                                         (0% del tráfico)        │
│                                [GREEN] Orders Service v1.5      │
│                                         (100% del tráfico) ✔    │
│                                                                 │
│  SI HAY PROBLEMAS → ROLLBACK INSTANTÁNEO:                       │
│                                                                 │
│  Internet ──▶ Load Balancer ──▶ [BLUE]  Orders Service v1.4     │
│                                         (100% del tráfico) ✔    │
│                                [GREEN] Orders Service v1.5      │
│                                         (0% del tráfico)        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Flujo completo de Blue/Green en ShopFast

```
  1. PREPARACIÓN:
     → Desplegar v1.5 en el entorno Green
     → Correr smoke tests contra Green (sin tráfico real)
     → Verificar health checks, métricas, conexiones a DB/Kafka

  2. SWITCH DE TRÁFICO:
     → Actualizar el Load Balancer: 100% → Green
     → Tiempo de switch: < 1 segundo (solo cambio de config del LB)
     → Blue queda en standby, no se apaga

  3. MONITOREO POST-SWITCH (ventana de observación: 15-30 min):
     → Error rate en Green vs baseline histórico de Blue
     → Latencia P99 en Green vs Blue
     → Alertas de negocio: órdenes creadas, pagos procesados
     → Logs de errores nuevos o inesperados

  4a. TODO OK → Blue se convierte en el nuevo standby para v1.6
  4b. PROBLEMA → rollback: apuntar el LB de vuelta a Blue (< 1 seg)
```

### Beneficios y desventajas de Blue/Green

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Zero downtime: el switch     │ Costo doble de infraestructura:  │
│ es instantáneo, sin          │ en todo momento hay dos entornos  │
│ requests perdidas.           │ de producción completos corriendo.│
│                              │                                  │
│ Rollback inmediato: volver   │ Migraciones de DB complejas:     │
│ a la versión anterior es     │ si v1.5 hace una migración de    │
│ un cambio de config, no      │ schema de DB, hacer rollback a   │
│ un nuevo deploy.             │ v1.4 puede romper la DB ya        │
│                              │ migrada.                         │
│ Entorno Green es idéntico    │                                  │
│ a producción — los smoke     │ No permite validar gradualmente: │
│ tests se corren contra        │ el switch es 0% ↔ 100%, sin     │
│ infraestructura real.        │ posibilidad de probar con el 5%  │
│                              │ de usuarios primero.             │
│ Simplicidad conceptual:      │                                  │
│ fácil de entender y          │ Estado de sesiones: si hay       │
│ de operar para el equipo.    │ sesiones activas en Blue durante │
│                              │ el switch, pueden perderse.      │
└──────────────────────────────┴──────────────────────────────────┘
```

---

### Estrategia 2 — Canary Testing (Canary Release)

**Español:**
**Canary Testing** (o Canary Release) es una estrategia en la que la nueva versión se despliega y recibe solo un **pequeño porcentaje del tráfico real** — por ejemplo, el 5%. El 95% restante continúa en la versión anterior. Si la nueva versión funciona correctamente con ese 5%, el porcentaje se va aumentando gradualmente (5% → 25% → 50% → 100%) hasta completar el despliegue. El nombre viene de los mineros que llevaban un canario a la mina: si el canario moría, había gases peligrosos — los primeros usuarios son el "canario" que detecta problemas antes de que afecten a todos.

**English:**
**Canary Testing** (or Canary Release) is a strategy where the new version is deployed and receives only a **small percentage of real traffic** — for example, 5%. The remaining 95% stays on the previous version. If the new version works correctly with that 5%, the percentage is gradually increased (5% → 25% → 50% → 100%) until the deployment is complete. The name comes from miners who brought a canary into the mine: if the canary died, there were dangerous gases — the first users are the "canary" that detects problems before they affect everyone.

```
┌─────────────────────────────────────────────────────────────────┐
│              CANARY TESTING — SHOPFAST                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FASE 1 — Deploy inicial (5% del tráfico):                      │
│                                                                 │
│  100 requests/seg entrantes                                     │
│       │                                                         │
│       ├──  5 req/seg ──▶ [CANARY]  Orders Service v1.5          │
│       └── 95 req/seg ──▶ [STABLE] Orders Service v1.4           │
│                                                                 │
│  Monitorear durante 30 min:                                     │
│  → Error rate canary vs stable: 0.01% vs 0.01% ✔               │
│  → Latencia P99 canary vs stable: 45ms vs 43ms ✔               │
│  → Órdenes creadas correctamente: sí ✔                          │
│                                                                 │
│  FASE 2 — Incremento gradual (25%):                             │
│                                                                 │
│       ├── 25 req/seg ──▶ [CANARY]  Orders Service v1.5          │
│       └── 75 req/seg ──▶ [STABLE] Orders Service v1.4           │
│                                                                 │
│  FASE 3 — Mayoría (50%) → FASE 4 — Completo (100%)             │
│                                                                 │
│  SI EN CUALQUIER FASE SE DETECTA PROBLEMA:                      │
│       ├──  0 req/seg ──▶ [CANARY]  v1.5 (rollback)              │
│       └── 100 req/seg──▶ [STABLE] v1.4 ✔                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Métricas clave a monitorear durante Canary

```
  Las métricas que determinan si el canary es seguro de avanzar:

  ┌──────────────────────────────────────────────────────────────┐
  │  Métrica              │ Canary v1.5  │ Stable v1.4 │ ¿OK?   │
  ├──────────────────────────────────────────────────────────────┤
  │  Error rate (5xx)     │  0.02%       │  0.01%      │  ✔     │
  │  Latencia P50         │  18ms        │  17ms       │  ✔     │
  │  Latencia P99         │  89ms        │  85ms       │  ✔     │
  │  Órdenes completadas  │  99.98%      │  99.99%     │  ✔     │
  │  Pagos procesados     │  100%        │  100%       │  ✔     │
  │  Memory usage         │  512MB       │  480MB      │  ✔     │
  │  CPU usage            │  45%         │  42%        │  ✔     │
  └──────────────────────────────────────────────────────────────┘

  Ejemplo de PROBLEMA detectado:
  │  Error rate (5xx)     │  2.50%  ❌   │  0.01%      │  ✗     │
  → Rollback automático: 0 tráfico al canary
  → Solo el 5% de usuarios fue afectado por el bug
```

### Beneficios y desventajas de Canary Testing

```
┌──────────────────────────────┬──────────────────────────────────┐
│  ✔ BENEFICIOS                │  ✗ DESVENTAJAS                   │
├──────────────────────────────┼──────────────────────────────────┤
│ Validación con tráfico real: │ Complejidad operacional mayor:   │
│ bugs que solo aparecen bajo  │ el LB o service mesh debe        │
│ carga real se detectan con   │ soportar traffic splitting con   │
│ impacto mínimo (5%).         │ porcentajes configurables         │
│                              │ (Istio, Nginx, AWS ALB).         │
│ Rollback fácil y rápido:     │                                  │
│ reducir el porcentaje del    │ Dos versiones en producción       │
│ canary a 0% es inmediato.    │ simultáneamente: los logs y las  │
│                              │ métricas deben diferenciarse     │
│ Despliegues sin ansiedad:    │ por versión para tener visión    │
│ el equipo gana confianza     │ clara de cada una.               │
│ progresivamente antes de     │                                  │
│ exponer al 100%.             │ APIs deben ser backward          │
│                              │ compatible: si v1.5 cambia un    │
│ Posibilidad de A/B testing   │ contrato de forma incompatible,  │
│ de features: comparar el     │ tener las dos versiones corriendo │
│ comportamiento de usuarios   │ simultáneamente causa errores.   │
│ en v1.4 vs v1.5 con datos    │                                  │
│ reales.                      │ Requiere observabilidad madura:  │
│                              │ sin buenas métricas por versión, │
│ Se puede automatizar con     │ es difícil saber si avanzar      │
│ criterios de avance          │ o hacer rollback.                │
│ (auto-promotion si           │                                  │
│ error rate < 0.1%).          │                                  │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 5. Comparación General: Blue/Green vs Canary

| Dimensión | Blue/Green | Canary |
|-----------|-----------|--------|
| Switch de tráfico | 0% → 100% (instantáneo) | Gradual (5% → 25% → 50% → 100%) |
| Validación con tráfico real | No antes del switch | Sí, desde el primer 5% |
| Rollback | Instantáneo (cambio de LB) | Rápido (reducir % a 0) |
| Costo de infraestructura | Alto (2 entornos completos) | Medio (% de recursos) |
| Impacto máximo de un bug | 100% (tras el switch) | 5% (si se detecta en fase 1) |
| Complejidad operacional | Media | Alta (traffic splitting) |
| DB migrations | Problemático | Problemático (igual) |
| Adecuado para | Releases estables con testing pre-prod exhaustivo | Features nuevas con riesgo o cambios de comportamiento |
| Herramientas | AWS CodeDeploy, Spinnaker | Istio, AWS ALB, Argo Rollouts |

### Cuándo usar cada estrategia en ShopFast

```
┌─────────────────────────────────────────────────────────────────┐
│           GUÍA DE DECISIÓN — SHOPFAST                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Usar BLUE/GREEN cuando:                                        │
│  → El cambio es una corrección de bug bien testeada             │
│  → El equipo necesita rollback instantáneo garantizado          │
│  → El costo de infraestructura doble es aceptable               │
│  → No hay migración de DB incompatible                          │
│                                                                 │
│  Usar CANARY cuando:                                            │
│  → Se lanza una feature nueva con comportamiento desconocido     │
│  → Se quiere medir el impacto en métricas de negocio (ej:       │
│    ¿el nuevo algoritmo de recomendaciones aumenta el CVR?)      │
│  → El servicio tiene alto tráfico y el riesgo de impactar       │
│    al 100% de usuarios es inaceptable                           │
│  → Se tiene observabilidad madura por versión                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "No amount of pre-production testing can fully replicate the complexity and volume of real production traffic. Blue/Green and Canary are not deployment tricks — they are testing strategies that use production itself as the final test environment, while limiting the blast radius of anything that goes wrong."

---

## 6. Contract Testing Solutions

### Pact

**Español:**
**Pact** es la herramienta de contract testing open-source más utilizada en el ecosistema de microservicios. Permite que los consumers de una API y los providers de esa API definan y verifiquen sus contratos de forma independiente. El consumer escribe tests que generan un archivo de contrato (`.pact.json`); el provider corre sus propios tests contra ese contrato para verificar que su implementación lo cumple. Pact soporta contratos HTTP/REST y mensajería asíncrona (eventos).

**English:**
**Pact** is the most widely used open-source contract testing tool in the microservices ecosystem. It allows API consumers and API providers to define and verify their contracts independently. The consumer writes tests that generate a contract file (`.pact.json`); the provider runs its own tests against that contract to verify its implementation complies. Pact supports HTTP/REST contracts and asynchronous messaging (events).

```
┌─────────────────────────────────────────────────────────────────┐
│                   PACT — SOPORTE DE LENGUAJES                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JVM:        Java, Kotlin, Scala, Groovy                        │
│  .NET:       C#, F#                                             │
│  JavaScript: Node.js, TypeScript (Jest, Mocha, Jasmine)         │
│  Ruby:       RSpec integration                                  │
│  Python:     pytest integration                                 │
│  Go:         pact-go                                            │
│  Swift / Objective-C: iOS native apps                           │
│  PHP:        pact-php                                           │
│  Rust:       pact-reference (implementación nativa del spec)    │
│  C++:        pact-cppsupport                                    │
│                                                                 │
│  Especificación: Pact Specification v3 / v4 (open standard)     │
│  Los archivos .pact.json son interoperables entre lenguajes.    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Repositorio:** [github.com/pact-foundation](https://github.com/pact-foundation)

---

### Spring Cloud Contract

**Español:**
**Spring Cloud Contract** es un proyecto umbrella open-source dentro del ecosistema Spring que contiene el sub-proyecto **Spring Cloud Contract Verifier** — orientado al desarrollo Consumer Driven Contract (CDC) en aplicaciones basadas en JVM. Los contratos se definen usando **Groovy DSL** o **YAML**, y Spring Cloud Contract genera automáticamente los stubs del provider (WireMock) y los tests de verificación del provider a partir de esos contratos. Está especialmente integrado con Spring Boot y Spring MVC/WebFlux.

**English:**
**Spring Cloud Contract** is an open-source umbrella project within the Spring ecosystem that contains the **Spring Cloud Contract Verifier** sub-project — oriented toward Consumer Driven Contract (CDC) development of JVM-based applications. Contracts are defined using **Groovy DSL** or **YAML**, and Spring Cloud Contract automatically generates provider stubs (WireMock) and provider verification tests from those contracts. It is especially well integrated with Spring Boot and Spring MVC/WebFlux.

```
┌─────────────────────────────────────────────────────────────────┐
│              SPRING CLOUD CONTRACT — FLUJO                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. El contrato se define en Groovy DSL o YAML:                 │
│                                                                 │
│  Contract.make {                                                │
│    request {                                                    │
│      method POST()                                              │
│      url '/payments/charge'                                     │
│      body(orderId: $(anyNonEmptyString()),                       │
│           amount:  $(anyPositiveDouble()))                       │
│    }                                                            │
│    response {                                                   │
│      status OK()                                                │
│      body(chargeId: $(anyNonEmptyString()),                      │
│           status:   "SUCCESS")                                  │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  2. Spring Cloud Contract genera automáticamente:               │
│     → Stubs WireMock para el consumer (Orders)                  │
│     → Tests de verificación para el provider (Payments)         │
│                                                                 │
│  Lenguajes soportados: todos los lenguajes de la JVM            │
│  (Java, Kotlin, Scala, Groovy)                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Repositorio:** [github.com/spring-cloud/spring-cloud-contract](https://github.com/spring-cloud/spring-cloud-contract)

---

### PactFlow

**Español:**
**PactFlow** es la oferta comercial construida sobre Pact que gestiona el **ciclo de vida completo de los contratos** definidos por herramientas de contract testing (Pact, Spring Cloud Contract, etc.). Provee un registro centralizado de contratos con versionado, la funcionalidad `can-i-deploy` para saber de forma segura si un servicio puede desplegarse sin romper sus contratos, y una integración nativa con pipelines de CI/CD. Incluye una UI para visualizar las relaciones entre consumers y providers, y el estado de verificación de cada contrato.

**English:**
**PactFlow** is the commercial offering built on top of Pact that manages the **complete lifecycle of contracts** defined by contract testing tools (Pact, Spring Cloud Contract, etc.). It provides a centralized contract registry with versioning, the `can-i-deploy` feature to safely determine if a service can be deployed without breaking its contracts, and native integration with CI/CD pipelines. It includes a UI to visualize consumer-provider relationships and the verification status of each contract.

```
┌─────────────────────────────────────────────────────────────────┐
│              PACTFLOW — CAPACIDADES CLAVE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✔ Registro centralizado de contratos con versionado            │
│  ✔ can-i-deploy: ¿puedo desplegar Orders v1.5 sin romper        │
│    los contratos con Payments, Notifications, Inventory?        │
│  ✔ Webhook notifications: notifica a los providers cuando       │
│    un consumer publica un nuevo contrato                         │
│  ✔ Network diagram: visualiza todas las dependencias            │
│    consumer-provider y el estado de verificación de cada una    │
│  ✔ Soporte para contratos HTTP y mensajería asíncrona           │
│  ✔ Compatible con cualquier herramienta de contract testing     │
│                                                                 │
│  Lenguajes soportados:                                          │
│  JVM (Java, Kotlin, Scala, Groovy), Ruby, Go, Node.js /         │
│  JavaScript, Python, Objective-C / Swift, .NET, PHP, C++, Rust  │
│                                                                 │
│  Modelo: SaaS (cloud) + opción on-premise (PactFlow Enterprise) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Comparativa de las tres soluciones

| Dimensión | Pact | Spring Cloud Contract | PactFlow |
|-----------|------|-----------------------|----------|
| Tipo | Open-source | Open-source | Comercial (SaaS) |
| Scope | Consumer + Provider testing | Consumer + Provider testing | Gestión del ciclo de vida |
| Lenguajes | Multi-lenguaje amplio | JVM principalmente | Todos los soportados por Pact |
| Definición del contrato | Código del consumer (test) | Groovy DSL o YAML | N/A (gestiona, no define) |
| Generación de stubs | Pact CLI / librería | Automática (WireMock) | N/A |
| Registro central | Pact Broker (auto-hosted) | Git + Artifactory | PactFlow (managed) |
| `can-i-deploy` | Via Pact Broker | No nativo | Nativo + UI |
| Integración CI/CD | CLI + plugins | Maven/Gradle plugin | Webhooks + API |
| Mejor para | Equipos multi-lenguaje | Equipos Spring/JVM | Organizaciones con muchos servicios |