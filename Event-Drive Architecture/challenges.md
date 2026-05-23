# Desafíos, Patrones y Herramientas / Challenges, Patterns & Tools

> Ver también / See also:
> - [Introducción y Definiciones](introduction.md)
> - [Beneficios](benefits.md)

---

## Desafíos de los Microservicios / Challenges of Microservices

### ⚡ 1. Microservicios como sistema altamente distribuido / Microservices as a Highly Distributed System

**Español:**
Los microservicios son por naturaleza un **sistema altamente distribuido**: múltiples procesos independientes corriendo en diferentes máquinas o contenedores, comunicándose a través de la red. Esto introduce toda una clase de problemas que no existen en un monolito, conocidos como las **"Falacias de la Computación Distribuida"** (Fallacies of Distributed Computing):

1. **La red es confiable** — Falso. Los paquetes se pierden, las conexiones se cortan.
2. **La latencia es cero** — Falso. Cada llamada de red tiene un costo en tiempo.
3. **El ancho de banda es infinito** — Falso. Hay límites de throughput.
4. **La red es segura** — Falso. El tráfico interno también puede ser interceptado.
5. **La topología no cambia** — Falso. Los servicios escalan, se mueven y se reinician.
6. **Hay un solo administrador** — Falso. Múltiples equipos gestionan distintos servicios.
7. **El costo de transporte es cero** — Falso. Serialización, deserialización y red tienen costo.
8. **La red es homogénea** — Falso. Distintos servicios pueden correr en distintas infraestructuras.

**English:**
Microservices are by nature a **highly distributed system**: multiple independent processes running on different machines or containers, communicating over the network. This introduces a whole class of problems that don't exist in a monolith, known as the **"Fallacies of Distributed Computing"**:

1. **The network is reliable** — False. Packets are lost, connections drop.
2. **Latency is zero** — False. Every network call has a time cost.
3. **Bandwidth is infinite** — False. There are throughput limits.
4. **The network is secure** — False. Internal traffic can also be intercepted.
5. **Topology doesn't change** — False. Services scale, move, and restart.
6. **There is one administrator** — False. Multiple teams manage different services.
7. **Transport cost is zero** — False. Serialization, deserialization, and network have a cost.
8. **The network is homogeneous** — False. Different services may run on different infrastructure.

**Implicaciones prácticas / Practical implications:**
```
En un monolito:
  serviceA.processOrder(data)  → llamada local, siempre funciona, microsegundos

En microservicios:
  POST http://order-service/orders  → puede fallar, puede tardar, puede timeout
  → Debes manejar: timeouts, reintentos, circuit breakers, fallbacks
  → Debes asumir: el servicio puede estar caído, lento o devolver datos corruptos
```

**Solución / Solution:** Diseñar para el fallo (Design for Failure): implementar timeouts en todas las llamadas de red, reintentos con backoff exponencial, Circuit Breaker, health checks y observabilidad completa.

---

### ❌ 2. Complejidad operacional / Operational Complexity

**Español:**
Gestionar decenas o cientos de servicios es un desafío enorme. Cada servicio necesita su propio pipeline de CI/CD, configuración de red, monitoreo, logging y escalado. Sin herramientas como Kubernetes, los equipos pasan más tiempo administrando infraestructura que desarrollando funcionalidades.

**English:**
Managing dozens or hundreds of services is a huge challenge. Each service needs its own CI/CD pipeline, network configuration, monitoring, logging, and scaling. Without tools like Kubernetes, teams spend more time managing infrastructure than developing features.

**Solución / Solution:** Kubernetes para orquestación, Helm para gestión de configuración, GitOps con ArgoCD para despliegues automatizados.

---

### ❌ 2. Latencia de red / Network Latency

**Español:**
La comunicación entre servicios ocurre **por red** (HTTP, gRPC, mensajes), lo que introduce latencia adicional. Una operación que en un monolito es una llamada a función en microsegundos, en microservicios puede requerir varias llamadas de red de 1-10ms cada una. En flujos complejos con muchas dependencias, esto se acumula.

**English:**
Communication between services happens **over the network** (HTTP, gRPC, messages), introducing additional latency. An operation that in a monolith is a microsecond function call can require several 1-10ms network calls in microservices. In complex flows with many dependencies, this accumulates.

**Solución / Solution:** gRPC en lugar de REST para comunicación síncrona, comunicación asíncrona via eventos para operaciones no críticas en tiempo real, caché con Redis.

---

### ❌ 3. Consistencia de datos distribuida / Distributed Data Consistency

**Español:**
Como cada servicio tiene su propia base de datos, **no existen transacciones ACID entre servicios**. Mantener la consistencia de datos a través de múltiples servicios es uno de los problemas más difíciles en sistemas distribuidos. Los datos pueden quedar en estados inconsistentes si un paso de una operación multi-servicio falla.

**English:**
Since each service has its own database, **there are no ACID transactions across services**. Maintaining data consistency across multiple services is one of the hardest problems in distributed systems. Data can end up in inconsistent states if a step of a multi-service operation fails.

**Solución / Solution:** Patrón Saga para transacciones distribuidas, consistencia eventual con compensaciones, patrón Outbox para garantizar entrega de eventos.

---

### ❌ 4. Testing distribuido / Distributed Testing

**Español:**
Probar la integración entre múltiples servicios es significativamente más complejo que en un monolito. Los entornos de test locales requieren correr múltiples servicios simultáneamente (Docker Compose). Los tests end-to-end son frágiles porque dependen de que todos los servicios estén disponibles.

**English:**
Testing integration between multiple services is significantly more complex than in a monolith. Local test environments require running multiple services simultaneously (Docker Compose). End-to-end tests are fragile because they depend on all services being available.

**Solución / Solution:** Consumer-Driven Contract Testing (Pact), tests de integración con Docker Compose, mocks y stubs para dependencias externas.

---

### ❌ 5. Debugging distribuido / Distributed Debugging

**Español:**
Cuando un error ocurre, el stack trace está **fragmentado entre múltiples servicios**. Una solicitud del usuario puede pasar por 5-10 servicios antes de fallar. Sin herramientas de tracing distribuido, es casi imposible reconstruir el flujo completo de una solicitud y localizar el origen del error.

**English:**
When an error occurs, the stack trace is **fragmented across multiple services**. A user request may pass through 5-10 services before failing. Without distributed tracing tools, it is nearly impossible to reconstruct the full flow of a request and locate the error's origin.

**Solución / Solution:** Jaeger o Zipkin para distributed tracing, correlación de logs con un `traceId` único por solicitud, ELK Stack para logging centralizado.

---

### ❌ 6. Overhead inicial / Initial Overhead

**Español:**
Configurar un proyecto de microservicios desde cero requiere: contenedores Docker, orquestación Kubernetes, API Gateway, service discovery, message broker, tracing, logging centralizado y más. Este overhead puede ser contraproducente para equipos pequeños o proyectos en fase temprana.

**English:**
Setting up a microservices project from scratch requires: Docker containers, Kubernetes orchestration, API Gateway, service discovery, message broker, tracing, centralized logging, and more. This overhead can be counterproductive for small teams or early-stage projects.

**Solución / Solution:** Empezar con un monolito bien estructurado y migrar gradualmente usando el patrón Strangler Fig cuando el sistema lo justifique.

---

### 🏢 8. Escalabilidad organizacional / Organization Scalability

**Español:**
Uno de los mayores beneficios de los microservicios es que permiten **escalar la organización**, no solo el sistema. A medida que una empresa crece y suma más desarrolladores, el monolito se convierte en un cuello de botella organizacional: muchos equipos pisan el mismo código, los merges son conflictivos y los despliegues requieren coordinación entre todos.

Los microservicios resuelven esto alineando la arquitectura técnica con la estructura de la organización (**Ley de Conway**):
> *"Las organizaciones producen diseños que son copia de sus estructuras de comunicación."* — Melvin Conway

Cada equipo es **dueño completo** de su servicio: lo diseña, desarrolla, despliega, monitorea y mantiene. No depende de ningún otro equipo para hacer su trabajo.

**English:**
One of the greatest benefits of microservices is that they allow **scaling the organization**, not just the system. As a company grows and adds more developers, the monolith becomes an organizational bottleneck: many teams touch the same code, merges are conflicting, and deployments require cross-team coordination.

Microservices solve this by aligning the technical architecture with the organization's structure (**Conway's Law**):
> *"Organizations produce designs that are copies of their communication structures."* — Melvin Conway

Each team has **full ownership** of its service: they design, develop, deploy, monitor, and maintain it. They don't depend on any other team to do their work.

**Comparación / Comparison:**
```
Organización con Monolito / Organization with Monolith:
  100 desarrolladores → 1 repositorio → conflictos constantes
  Deploy requiere coordinar a todos los equipos
  Un equipo bloquea a todos los demás

Organización con Microservicios / Organization with Microservices:
  100 desarrolladores → 15 equipos de ~7 personas → 15 servicios independientes
  Cada equipo despliega cuando quiere, sin coordinación
  Un equipo no bloquea a los demás
```

**Estructura típica / Typical Structure:**
```
Equipo Payments   → Payment Service    (deploy independiente ✅)
Equipo Orders     → Order Service      (deploy independiente ✅)
Equipo Catalog    → Product Service    (deploy independiente ✅)
Equipo Identity   → Auth Service       (deploy independiente ✅)
Equipo Platform   → Infraestructura, Kubernetes, CI/CD compartido
```

**Desafío organizacional / Organizational Challenge:**
Sin embargo, escalar la organización también introduce nuevos retos:
- **Duplicación de esfuerzo:** Cada equipo puede resolver el mismo problema de forma diferente (logging, auth, manejo de errores).
- **Estándares inconsistentes:** Sin gobernanza, cada servicio puede tener su propio estilo de API, versionado y documentación.
- **Comunicación entre equipos:** Los contratos entre servicios (APIs y eventos) deben ser acordados y versionados cuidadosamente.

**Solución / Solution:** Crear un equipo de plataforma (Platform Team) que provea herramientas, templates y estándares compartidos. Usar Inner Source (patrones de open source internamente), API design guidelines y contract testing (Pact) para mantener consistencia sin bloquear la autonomía de los equipos.

---

## Desafíos de la Arquitectura Orientada a Eventos / Challenges of EDA

### ❌ 1. Debugging asíncrono / Asynchronous Debugging

**Español:**
El flujo asíncrono hace **difícil rastrear el camino de un evento** a través del sistema. Un evento publicado puede tardar segundos o minutos en procesarse. Si un consumidor falla, el error puede aparecer mucho después del evento original y en un servicio completamente diferente, complicando enormemente el debugging.

**English:**
The asynchronous flow makes it **hard to trace the path of an event** through the system. A published event may take seconds or minutes to be processed. If a consumer fails, the error may appear long after the original event and in a completely different service, greatly complicating debugging.

**Solución / Solution:** Distributed tracing (Jaeger), correlación de `traceId` en todos los eventos, Dead Letter Queues (DLQ) para eventos que fallaron.

---

### ❌ 2. Consistencia eventual / Eventual Consistency

**Español:**
En EDA, los datos **no son consistentes de forma inmediata**. Después de que un productor publica un evento, los consumidores tardan un tiempo (milisegundos a segundos) en procesarlo y actualizar sus propios datos. Durante este período, diferentes partes del sistema pueden tener visiones inconsistentes del estado.

**English:**
In EDA, data is **not immediately consistent**. After a producer publishes an event, consumers take time (milliseconds to seconds) to process it and update their own data. During this window, different parts of the system may have inconsistent views of the state.

**Ejemplo / Example:**
```
Usuario hace una compra:
1. Order Service publica "OrderCreated" → responde 200 OK al usuario
2. El usuario ve su orden como "Confirmada" ✅
3. Inventory Service aún NO procesó el evento (está ocupado)
4. Si el usuario consulta el stock en ese momento → puede ver stock incorrecto
5. 500ms después: Inventory Service procesa → sistema consistente ✅
```

**Solución / Solution:** Diseñar la UI para tolerar eventual consistency, usar spinners/estados pendientes, informar al usuario que la operación está "en proceso".

---

### ❌ 3. Orden de eventos / Event Ordering

**Español:**
En sistemas distribuidos, **no se garantiza el orden de llegada de los eventos** a los consumidores, especialmente con múltiples particiones en Kafka o múltiples instancias de consumidores. Un evento "OrderUpdated" podría procesarse antes que el "OrderCreated" original, causando errores de lógica.

**English:**
In distributed systems, **the order of event arrival at consumers is not guaranteed**, especially with multiple Kafka partitions or multiple consumer instances. An "OrderUpdated" event could be processed before the original "OrderCreated", causing logic errors.

**Solución / Solution:** Usar partition keys en Kafka para garantizar orden dentro de una partición, incluir número de versión/secuencia en los eventos, diseñar consumidores idempotentes.

---

### ❌ 4. Duplicación de eventos / Event Duplication

**Español:**
Los message brokers generalmente ofrecen garantía **"at-least-once delivery"** (entrega al menos una vez). Esto significa que un evento puede ser entregado **más de una vez** al consumidor (por reintentos, rebalanceos de partición, etc.). Si el consumidor no es idempotente, puede procesar el mismo evento dos veces causando efectos duplicados (cobrar dos veces, crear dos registros, etc.).

**English:**
Message brokers generally offer **"at-least-once delivery"** guarantee. This means an event may be delivered **more than once** to the consumer (due to retries, partition rebalancing, etc.). If the consumer is not idempotent, it may process the same event twice causing duplicate effects (charging twice, creating two records, etc.).

**Solución / Solution:** Diseñar consumidores idempotentes, usar el patrón Inbox para deduplicación, verificar si el evento ya fue procesado antes de actuar.

---

## Desafíos del Monolito / Challenges of Monolithic

### ❌ 1. Escalabilidad limitada / Limited Scalability

**Español:**
Para escalar el sistema, debes escalar **toda** la aplicación, incluso si solo un módulo tiene alta demanda. Esto desperdicia recursos porque se replica código y capacidad que no la necesita. No puedes escalar el módulo de pagos sin escalar también usuarios, productos y notificaciones.

**English:**
To scale the system, you must scale the **entire** application, even if only one module has high demand. This wastes resources because code and capacity that doesn't need scaling is replicated. You can't scale the payments module without also scaling users, products, and notifications.

---

### ❌ 2. Despliegue de alto riesgo / High-Risk Deployment

**Español:**
Un cambio en cualquier módulo requiere redesplegar **toda** la aplicación, lo que implica downtime potencial y alto riesgo. Si el despliegue falla, **todo** el sistema puede quedar inoperativo. Los equipos tienden a desplegar con menos frecuencia por el miedo al riesgo, acumulando más cambios por deploy y aumentando el riesgo aún más.

**English:**
A change in any module requires redeploying the **entire** application, implying potential downtime and high risk. If the deployment fails, the **entire** system may become inoperative. Teams tend to deploy less frequently due to risk fear, accumulating more changes per deploy and increasing risk even further.

---

### ❌ 3. Acoplamiento fuerte y deuda técnica / Tight Coupling & Technical Debt

**Español:**
Con el tiempo, los módulos del monolito se vuelven **interdependientes de forma no intencionada**. Los desarrolladores toman atajos, acceden directamente a las tablas de otros módulos, comparten variables globales y crean dependencias ocultas. El resultado es código espagueti imposible de modificar sin romper algo.

**English:**
Over time, monolith modules become **unintentionally interdependent**. Developers take shortcuts, directly access other modules' tables, share global variables, and create hidden dependencies. The result is spaghetti code impossible to modify without breaking something.

---

### ❌ 4. Bloqueo tecnológico / Technology Lock-in

**Español:**
Todo el sistema está atado al mismo lenguaje, versión del framework y base de datos. Si aparece una tecnología mejor para un módulo específico, migrar es extremadamente costoso porque requiere cambiar todo el sistema. El monolito "envejece" junto con su stack tecnológico.

**English:**
The entire system is tied to the same language, framework version, and database. If a better technology appears for a specific module, migrating is extremely costly because it requires changing the whole system. The monolith "ages" together with its technology stack.

---

### ❌ 5. Coordinación entre equipos / Team Coordination

**Español:**
Múltiples equipos trabajando en el mismo codebase generan **conflictos de merge frecuentes**, necesidad de coordinación para despliegues y riesgo de que los cambios de un equipo rompan el trabajo de otro. El ciclo de desarrollo se vuelve lento y frustrante a medida que el equipo crece.

**English:**
Multiple teams working on the same codebase generate **frequent merge conflicts**, need for deployment coordination, and risk of one team's changes breaking another's work. The development cycle becomes slow and frustrating as the team grows.

---

## Patrones para Resolver los Desafíos / Patterns to Address the Challenges

---

### 1. Saga — Transacciones distribuidas / Distributed Transactions

**Español:**
El patrón Saga resuelve el problema de las transacciones distribuidas. Como cada microservicio tiene su propia base de datos, no se pueden usar transacciones ACID. Una Saga divide la transacción en una secuencia de transacciones locales. Si algún paso falla, se ejecutan **transacciones de compensación** para deshacer los cambios anteriores.

- **Choreography:** Cada servicio publica un evento al completar su paso. Sin coordinador central.
- **Orchestration:** Un servicio orquestador central dirige a cada servicio qué hacer y cuándo.

**English:**
The Saga pattern solves the distributed transaction problem. Since each microservice has its own database, ACID transactions are not possible. A Saga splits the transaction into local transactions. If any step fails, **compensating transactions** undo previous changes.

- **Choreography:** Each service publishes an event upon completing its step. No central coordinator.
- **Orchestration:** A central orchestrator service directs each service on what to do and when.

**Ejemplo / Example:**
```
Flujo de compra / Purchase flow:
1. Order Service     → crea orden           (OrderCreated)
2. Payment Service   → cobra pago           (PaymentProcessed)
3. Inventory Service → reserva stock        (StockReserved)
4. Shipping Service  → crea envío           (ShipmentCreated)

Si el pago falla en el paso 2 / If payment fails at step 2:
← Order Service cancela la orden (transacción de compensación)
```

---

### 2. CQRS — Separar lecturas de escrituras / Separate Reads from Writes

**Español:**
CQRS (Command Query Responsibility Segregation) separa el modelo de **escritura** (Commands) del modelo de **lectura** (Queries) en dos modelos distintos, incluso con bases de datos separadas. El modelo de escritura maneja la lógica de negocio y la consistencia; el modelo de lectura está optimizado para consultas rápidas y puede ser desnormalizado.

**English:**
CQRS separates the **write** model (Commands) from the **read** model (Queries) into two distinct models, even with separate databases. The write model handles business logic and consistency; the read model is optimized for fast queries and can be denormalized.

**Ejemplo / Example:**
```
Command side (escritura / write):
  POST /orders → OrderService → PostgreSQL (ACID, normalizado)
                              → publica evento OrderCreated

Query side (lectura / read):
  GET /orders/dashboard → ReadService → Elasticsearch (desnormalizado, rápido)
                                      → respuesta en <10ms
```

---

### 3. Event Sourcing — Historial de eventos como fuente de verdad

**Español:**
En lugar de guardar solo el estado actual, Event Sourcing guarda **todos los eventos** que llevaron a ese estado. El estado se reconstruye reproduciendo eventos en orden. Proporciona un log de auditoría completo y permite reconstruir el estado en cualquier punto del tiempo.

**English:**
Instead of storing only the current state, Event Sourcing stores **all events** that led to that state. The state is reconstructed by replaying events in order. It provides a complete audit log and allows rebuilding state at any point in time.

**Ejemplo / Example:**
```
Cuenta bancaria / Bank account:
  1. AccountOpened    { balance: 0 }
  2. MoneyDeposited   { amount: 1000 }
  3. MoneyWithdrawn   { amount: 200 }
  4. MoneyDeposited   { amount: 500 }

Estado actual: 0 + 1000 - 200 + 500 = $1300
```

---

### 4. Circuit Breaker — Prevenir fallos en cascada / Prevent Cascading Failures

**Español:**
El Circuit Breaker evita que un fallo en un servicio destruya todo el sistema. Si un servicio destino falla repetidamente, el Circuit Breaker "abre el circuito" y devuelve respuestas de error inmediatas (o un fallback) sin intentar llamar al servicio caído.

**3 estados / 3 states:**
- **Closed (Cerrado):** Todo funciona. Las llamadas pasan normalmente.
- **Open (Abierto):** El servicio falla. Las llamadas se bloquean inmediatamente.
- **Half-Open (Semiabierto):** Se permite una llamada de prueba para verificar recuperación.

**English:**
The Circuit Breaker prevents a service failure from destroying the whole system. If a target service fails repeatedly, the Circuit Breaker "opens the circuit" and returns immediate error responses (or a fallback) without calling the down service.

**Ejemplo / Example:**
```
Si Payment Service falla 5 veces seguidas:
  Circuit ABIERTO → respuesta inmediata: "Servicio no disponible"
  (sin esperar timeout de red de 30s)

Después de 30 segundos → prueba una llamada (Half-Open)
  Si exitosa → Circuit CERRADO ✅
```

---

### 5. API Gateway — Punto de entrada único / Single Entry Point

**Español:**
El API Gateway es el punto de entrada único para todos los clientes. Enruta solicitudes al microservicio correcto y maneja autenticación, autorización, rate limiting, logging y SSL termination. Evita que los clientes conozcan la dirección interna de cada servicio.

**English:**
The API Gateway is the single entry point for all clients. It routes requests to the correct microservice and handles authentication, authorization, rate limiting, logging, and SSL termination. It prevents clients from knowing the internal address of each service.

**Ejemplo / Example:**
```
Cliente → API Gateway (JWT auth, rate limit, SSL)
              ├── /api/users/*    → User Service    :3001
              ├── /api/orders/*   → Order Service   :3002
              ├── /api/products/* → Product Service :3003
              └── /api/payments/* → Payment Service :3004
```

---

### 6. Strangler Fig — Migración gradual / Gradual Migration

**Español:**
Estrategia para migrar de un monolito a microservicios sin una reescritura completa. Se "estrangula" el monolito extrayendo funcionalidades a nuevos microservicios una por una. El API Gateway enruta el tráfico según la funcionalidad. Con el tiempo, el monolito queda vacío.

**English:**
Strategy to migrate from a monolith to microservices without a full rewrite. The monolith is "strangled" by extracting functionalities into new microservices one by one. The API Gateway routes traffic based on functionality. Over time, the monolith becomes empty.

**Ejemplo / Example:**
```
Fase 1: Todo en el monolito
Fase 2: API Gateway → /payments/* → Payment Microservice ✅ | resto → Monolith
Fase 3: API Gateway → /payments/* → Payment Microservice ✅
                   → /orders/*   → Order Microservice   ✅ | resto → Monolith
... hasta que el monolito desaparece
```

---

### 7. Database per Service — Base de datos por servicio / DB per Service

**Español:**
Cada microservicio tiene su propia base de datos **completamente privada**. Ningún otro servicio puede acceder a ella directamente. Garantiza el desacoplamiento total. Cada servicio elige el tipo de DB más adecuado (SQL, NoSQL, grafo, caché).

**English:**
Each microservice has its own **completely private** database. No other service can access it directly. Guarantees full decoupling. Each service chooses the most suitable DB type (SQL, NoSQL, graph, cache).

**Ejemplo / Example:**
```
User Service    → PostgreSQL    (relacional / relational)
Product Service → MongoDB       (documentos flexibles / flexible documents)
Cart Service    → Redis         (caché y sesiones / cache and sessions)
Search Service  → Elasticsearch (full-text search)
Graph Service   → Neo4j         (relaciones complejas / complex relationships)
```

---

### 8. Inbox / Outbox — Garantía de entrega / Delivery Guarantee

**Español:**
Garantiza que los eventos se publiquen **exactamente una vez**, incluso si el servicio falla. El servicio guarda el evento en una tabla `outbox` en la **misma transacción** que modifica los datos. Un proceso separado (relay) publica los eventos al broker. El receptor usa una tabla `inbox` para deduplicación.

**English:**
Guarantees events are published **exactly once**, even if the service crashes. The service saves the event to an `outbox` table in the **same transaction** that modifies data. A separate process (relay) publishes events to the broker. The receiver uses an `inbox` table for deduplication.

**Ejemplo / Example:**
```
BEGIN TRANSACTION
  INSERT INTO orders    VALUES (1, 'created')
  INSERT INTO outbox    VALUES ('OrderCreated', '{id:1}')
COMMIT

Relay: Lee outbox → publica a Kafka → marca como enviado

Consumer: Recibe evento → verifica inbox (ya procesado?)
  → Si sí: ignora (idempotente)
  → Si no: procesa + guarda en inbox
```

---

## Herramientas y Tecnologías / Tools & Technologies

| Categoría / Category | Herramientas / Tools | Uso / Use |
|---|---|---|
| **Message Brokers** | Apache Kafka, RabbitMQ, AWS SQS/SNS, Azure Service Bus | Comunicación asíncrona entre servicios |
| **Containerización** | Docker, Podman | Empaquetar servicios en contenedores portables |
| **Orquestación** | Kubernetes (K8s), Docker Swarm | Gestionar y escalar contenedores en producción |
| **API Gateway** | Kong, AWS API Gateway, NGINX, Traefik | Punto de entrada único, auth, rate limiting |
| **Service Mesh** | Istio, Linkerd | Seguridad, observabilidad y control del tráfico entre servicios |
| **Tracing distribuido** | Jaeger, Zipkin, AWS X-Ray | Rastrear solicitudes a través de múltiples servicios |
| **Logging centralizado** | ELK Stack, Grafana Loki, Datadog | Agregar logs de todos los servicios en un solo lugar |
| **Monitoreo / Metrics** | Prometheus, Grafana, Datadog | Métricas de rendimiento y alertas |
| **Service Discovery** | Consul, Eureka, Kubernetes DNS | Localizar servicios dinámicamente en la red |
| **CI/CD** | GitHub Actions, Jenkins, GitLab CI, ArgoCD | Automatizar build, test y deploy por servicio |
| **Contract Testing** | Pact | Verificar contratos de API entre productor y consumidor |
| **Circuit Breaker** | Resilience4j, Hystrix, Istio | Implementar el patrón Circuit Breaker |
| **Config Management** | HashiCorp Vault, AWS Secrets Manager, Consul | Gestionar configuración y secretos por servicio |
