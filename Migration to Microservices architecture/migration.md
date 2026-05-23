# Migración a Microservicios / Migration to Microservices

> Ver también / See also:
> - [Introducción y Definiciones](introduction.md)
> - [Beneficios](benefits.md)
> - [Desafíos, Patrones y Herramientas](challenges.md)

---

## 1. Límites de Microservicios — Principios fundamentales / Microservices Boundaries — Core Principles

### ¿Qué es un límite de servicio? / What is a Service Boundary?

**Español:**
Un **límite de servicio** (service boundary) define qué responsabilidades pertenecen a un microservicio y cuáles no. Es la línea que separa un servicio de otro. Definir bien estos límites es la decisión más importante al diseñar una arquitectura de microservicios. Un límite mal definido genera servicios demasiado acoplados (que se comportan como un monolito distribuido) o demasiado pequeños (que añaden complejidad sin valor).

**English:**
A **service boundary** defines which responsibilities belong to a microservice and which do not. It is the line that separates one service from another. Defining these boundaries well is the most important decision when designing a microservices architecture. A poorly defined boundary creates overly coupled services (behaving like a distributed monolith) or overly small services (adding complexity without value).

---

### Domain-Driven Design (DDD) y Bounded Contexts

**Español:**
La herramienta más poderosa para definir límites es el **Domain-Driven Design (DDD)**, específicamente el concepto de **Bounded Context**. Un Bounded Context es una región del dominio de negocio donde un modelo, lenguaje y reglas específicas aplican de forma consistente. Cada microservicio debería corresponder (idealmente) a un Bounded Context.

**English:**
The most powerful tool for defining boundaries is **Domain-Driven Design (DDD)**, specifically the concept of **Bounded Context**. A Bounded Context is a region of the business domain where a specific model, language, and rules apply consistently. Each microservice should (ideally) correspond to one Bounded Context.

**Ejemplo / Example:**
```
Dominio: E-commerce

Bounded Context 1: Catálogo (Catalog)
  → Entidades: Product, Category, Price
  → Lenguaje: "producto", "precio de lista", "categoría"

Bounded Context 2: Pedidos (Orders)
  → Entidades: Order, OrderItem, ShippingAddress
  → Lenguaje: "pedido", "ítem", "dirección de entrega"
  → Nota: "Product" aquí solo es un ID + nombre, no el mismo objeto del Catálogo

Bounded Context 3: Pagos (Payments)
  → Entidades: Payment, Invoice, Refund
  → Lenguaje: "pago", "factura", "reembolso"
```

---

### Principios para definir buenos límites / Principles for Defining Good Boundaries

#### ✅ Alta cohesión / High Cohesion
**ES:** Todo lo que cambia junto debe estar junto. Si dos funcionalidades siempre se modifican al mismo tiempo, probablemente pertenecen al mismo servicio.
**EN:** Everything that changes together should stay together. If two functionalities are always modified at the same time, they probably belong to the same service.

#### ✅ Bajo acoplamiento / Loose Coupling
**ES:** Los servicios deben depender lo menos posible entre sí. Si cambiar un servicio siempre requiere cambiar otro, el límite está mal definido.
**EN:** Services should depend on each other as little as possible. If changing one service always requires changing another, the boundary is poorly defined.

#### ✅ Organizado por negocio, no por tecnología / Organized by Business, Not Technology
**ES:** No dividir por capas técnicas (un servicio para la DB, otro para la lógica, otro para la UI). Dividir por capacidades de negocio (Payments, Orders, Users).
**EN:** Don't split by technical layers (one service for DB, another for logic, another for UI). Split by business capabilities (Payments, Orders, Users).

#### ✅ Responsabilidad única / Single Responsibility
**ES:** Cada servicio debe hacer una sola cosa y hacerla bien. Si el nombre del servicio requiere la palabra "y" (ej. "Orders**And**Payments"), probablemente son dos servicios.
**EN:** Each service should do one thing and do it well. If the service name requires the word "and" (e.g., "Orders**And**Payments"), it's probably two services.

#### ✅ Propiedad de sus datos / Data Ownership
**ES:** Cada servicio es el **único dueño** de sus datos. Ningún otro servicio puede leer o escribir directamente en su base de datos. La comunicación de datos se hace solo por API o eventos.
**EN:** Each service is the **sole owner** of its data. No other service can read or write directly to its database. Data communication happens only through APIs or events.

#### ⚠️ Evitar el monolito distribuido / Avoid the Distributed Monolith
**ES:** El peor resultado posible: múltiples servicios que están tan acoplados que deben desplegarse juntos, o que se llaman de forma sincrónica en largas cadenas. Tiene la complejidad de los microservicios **y** las limitaciones del monolito.
**EN:** The worst possible outcome: multiple services so coupled they must be deployed together, or that call each other synchronously in long chains. It has the complexity of microservices **and** the limitations of the monolith.

```
❌ Monolito distribuido (MAL / BAD):
   Request → Service A → Service B → Service C → Service D → DB
   (si D falla, toda la cadena falla. Igual que un monolito, pero más lento)

✅ Microservicios bien definidos (BIEN / GOOD):
   Request → Service A (completa su trabajo de forma independiente)
           → publica evento → Service B reacciona de forma asíncrona
```

---

## 2. Descomposición del Monolito / Decomposition of a Monolithic Application

**Español:**
Descomponer un monolito significa identificar y extraer funcionalidades del monolito para convertirlas en microservicios independientes. No es simplemente "cortar el código en pedazos" — requiere un análisis cuidadoso del dominio de negocio, las dependencias entre módulos y la capacidad del equipo.

**English:**
Decomposing a monolith means identifying and extracting functionalities from the monolith to convert them into independent microservices. It's not simply "cutting the code into pieces" — it requires careful analysis of the business domain, module dependencies, and team capacity.

---

### Estrategias de descomposición / Decomposition Strategies

#### 📦 Por capacidad de negocio / By Business Capability
**ES:** Identificar las **capacidades de negocio** de la organización (lo que el negocio hace) y crear un servicio por cada una. Es la estrategia más recomendada porque los servicios resultantes son estables: las capacidades de negocio cambian menos que la tecnología.

**EN:** Identify the **business capabilities** of the organization (what the business does) and create one service per capability. This is the most recommended strategy because the resulting services are stable: business capabilities change less than technology.

```
Capacidades de negocio de un e-commerce:
  → Gestión de usuarios       → User Service
  → Catálogo de productos     → Product Service
  → Gestión de pedidos        → Order Service
  → Procesamiento de pagos    → Payment Service
  → Gestión de inventario     → Inventory Service
  → Notificaciones            → Notification Service
  → Envíos y logística        → Shipping Service
```

#### 📦 Por subdominio (DDD) / By Subdomain (DDD)
**ES:** Usar Domain-Driven Design para identificar subdominios del negocio: **Core Domain** (lo que diferencia al negocio), **Supporting Domain** (apoya al core) y **Generic Domain** (funcionalidad genérica que podría ser un servicio de terceros).

**EN:** Use Domain-Driven Design to identify business subdomains: **Core Domain** (what differentiates the business), **Supporting Domain** (supports the core), and **Generic Domain** (generic functionality that could be a third-party service).

```
Core Domain (máxima inversión):
  → Recommendation Engine, Pricing Algorithm

Supporting Domain:
  → Order Management, Inventory

Generic Domain (usar SaaS si es posible):
  → Authentication → usar Auth0 o Keycloak
  → Email → usar SendGrid o AWS SES
  → Payments → usar Stripe o PayPal
```

#### 📦 Por verbos / By Verbs (Use Cases)
**ES:** Identificar las acciones principales del sistema (casos de uso) y agrupar las relacionadas en un servicio. Útil cuando el dominio no está bien definido todavía.

**EN:** Identify the system's main actions (use cases) and group related ones into a service. Useful when the domain is not yet well defined.

```
Acciones de usuario / User actions:
  → Register, Login, UpdateProfile  → User Service
  → BrowseProducts, SearchProducts  → Catalog Service
  → AddToCart, Checkout             → Order Service
  → MakePayment, Refund             → Payment Service
```

---

### Identificar dependencias / Identifying Dependencies

**Español:**
Antes de extraer un módulo, es crucial entender sus **dependencias**: ¿qué otros módulos llama? ¿qué otros módulos lo llaman? ¿comparte tablas de base de datos? Un módulo con muchas dependencias bidireccionales es difícil de extraer y puede ser señal de que los límites deben redefinirse.

**English:**
Before extracting a module, it's crucial to understand its **dependencies**: what other modules does it call? What other modules call it? Does it share database tables? A module with many bidirectional dependencies is hard to extract and may signal that boundaries need to be redefined.

```
Herramienta útil: Dependency Matrix / Matriz de dependencias

          | Users | Orders | Payments | Inventory |
----------|-------|--------|----------|-----------|
Users     |   -   |   →    |    →     |           |
Orders    |   ←   |   -    |    →     |    →      |
Payments  |   ←   |   ←    |    -     |           |
Inventory |       |   ←    |          |    -      |

→ Orders tiene dependencias con todos: extraer al último
→ Inventory solo depende de Orders: buen candidato para extraer primero
```

---

## 3. Migración a Microservicios — Pasos, Consejos y Patrones / Migration Steps, Tips & Patterns

### ¿Cuándo migrar? / When to Migrate?

**Español:**
No migres solo porque es la tendencia. Migra cuando el monolito cause **problemas reales y medibles**:

**English:**
Don't migrate just because it's trendy. Migrate when the monolith causes **real and measurable problems**:

| Señal de alerta / Warning Sign | Descripción |
|---|---|
| Deployments lentos y riesgosos | El deploy tarda horas y requiere coordinación de todos los equipos |
| Equipos bloqueados entre sí | El equipo A no puede desplegar porque el equipo B tiene cambios pendientes |
| Escalabilidad ineficiente | Debes escalar toda la app cuando solo un módulo tiene alta carga |
| Builds que tardan >10-15 min | El ciclo de desarrollo se vuelve lento y frustrante |
| Alta tasa de bugs en deploys | Cambios pequeños rompen cosas no relacionadas |
| Onboarding muy lento | Los nuevos desarrolladores tardan meses en entender el sistema |

---

### Pasos de migración / Migration Steps

#### Paso 1: Entender el monolito / Understand the Monolith
**ES:** Antes de tocar el código, documentar la arquitectura actual: módulos, dependencias, flujos de datos y puntos de dolor. Usar herramientas de análisis estático de código para descubrir dependencias ocultas.
**EN:** Before touching the code, document the current architecture: modules, dependencies, data flows, and pain points. Use static code analysis tools to discover hidden dependencies.

#### Paso 2: Modularizar el monolito / Modularize the Monolith
**ES:** Si el monolito no tiene módulos bien definidos, el primer paso es refactorizarlo internamente para crear separaciones claras entre módulos (sin cambiar la infraestructura). Esto hace la futura extracción mucho más fácil.
**EN:** If the monolith doesn't have well-defined modules, the first step is to internally refactor it to create clear separations between modules (without changing the infrastructure). This makes future extraction much easier.

```
Monolito sin estructura:           Monolito modular (paso previo):
  src/                               src/
    controllers/                       modules/
    models/                              users/
    services/                            orders/
    utils/                               payments/
                                         inventory/
```

#### Paso 3: Identificar el primer servicio a extraer / Identify the First Service to Extract
**ES:** Elegir el primer servicio con cuidado. Criterios recomendados:
**EN:** Choose the first service carefully. Recommended criteria:

```
✅ Buen primer candidato:          ❌ Mal primer candidato:
  - Pocos dependencias              - Muchas dependencias bidireccionales
  - Dominio bien definido           - Comparte muchas tablas con otros módulos
  - Equipo claro dueño del módulo   - Lógica de negocio central y compleja
  - Bajo riesgo (no crítico)        - Alta criticidad (pagos, auth)

Ejemplos buenos primeros servicios:
  → Notification Service (solo recibe eventos, sin dependencias hacia otros)
  → Email Service        (funcionalidad periférica y bien aislada)
  → Report Service       (solo lee datos, no los modifica)
```

#### Paso 4: Aplicar el patrón Strangler Fig
**ES:** Extraer el servicio gradualmente sin apagar el monolito. El API Gateway redirige el tráfico al nuevo servicio para esa funcionalidad, mientras el monolito sigue manejando el resto.
**EN:** Extract the service gradually without shutting down the monolith. The API Gateway redirects traffic to the new service for that functionality, while the monolith continues handling the rest.

```
Antes / Before:
  Todos los requests → Monolito

Durante la migración / During migration:
  /api/notifications/* → Notification Service (nuevo ✅)
  /api/*               → Monolito (todo lo demás)

Después de varios ciclos / After several cycles:
  /api/notifications/* → Notification Service ✅
  /api/emails/*        → Email Service        ✅
  /api/reports/*       → Report Service       ✅
  /api/*               → Monolito (cada vez más pequeño)
```

#### Paso 5: Separar la base de datos / Separate the Database
**ES:** Este es el paso más difícil. El monolito y el nuevo servicio pueden compartir temporalmente la base de datos (Database Sharing antipattern) durante la migración, pero el objetivo final es que cada servicio tenga su propia DB. La transición se hace en fases:
**EN:** This is the hardest step. The monolith and the new service can temporarily share the database (Database Sharing antipattern) during migration, but the end goal is each service having its own DB. The transition happens in phases:

```
Fase A: Servicio nuevo comparte DB del monolito (temporal, no ideal)
  New Service ──→ Monolith DB

Fase B: Separar las tablas del nuevo servicio
  New Service ──→ New Service DB (tablas propias)
  New Service ──→ Monolith DB   (solo las tablas que aún comparte, via API)

Fase C: Eliminar dependencia con DB del monolito
  New Service ──→ New Service DB ✅ (completamente independiente)
  Monolito    ──→ Monolith DB    ✅
```

#### Paso 6: Repetir con el siguiente servicio / Repeat with Next Service
**ES:** Una vez que el primer servicio está estable y completamente independiente, repetir el proceso con el siguiente candidato. Priorizar los servicios que más dolor causan (alta carga, muchos bugs, bloqueo de equipos).
**EN:** Once the first service is stable and fully independent, repeat the process with the next candidate. Prioritize services causing the most pain (high load, many bugs, team blocking).

---

### Consejos importantes / Important Tips

| Consejo / Tip | Descripción ES | Description EN |
|---|---|---|
| **No big bang** | Nunca reescribir todo desde cero de una vez. Es el camino más seguro al fracaso. | Never rewrite everything from scratch at once. It's the safest path to failure. |
| **Migrar datos con cuidado** | Separar una base de datos compartida es el paso más riesgoso. Hacerlo con doble-escritura y verificación. | Separating a shared database is the riskiest step. Do it with dual-write and verification. |
| **Mantener el monolito funcional** | El monolito debe seguir funcionando durante toda la migración. Nunca dejarlo en estado roto. | The monolith must keep working throughout the migration. Never leave it in a broken state. |
| **Medir antes y después** | Definir métricas de éxito antes de migrar (latencia, error rate, deploy time) para verificar que la migración mejora las cosas. | Define success metrics before migrating (latency, error rate, deploy time) to verify the migration improves things. |
| **Empezar por la periferia** | Extraer primero los servicios más externos y con menos dependencias. Dejar el núcleo del negocio para el final. | Extract the most external, least-dependent services first. Leave the business core for last. |
| **Automatizar desde el inicio** | Cada nuevo servicio debe tener CI/CD automatizado desde el día 1. No hacerlo manualmente. | Every new service must have automated CI/CD from day 1. Don't do it manually. |

---

### Patrones de migración / Migration Patterns

| Patrón / Pattern | Cuándo usarlo / When to Use |
|---|---|
| **Strangler Fig** | Migración gradual — extraer funcionalidades una por una redirigiendo tráfico via API Gateway |
| **Branch by Abstraction** | Cuando no puedes usar un API Gateway — crear una abstracción interna para redirigir llamadas |
| **Parallel Run** | Correr el monolito y el nuevo servicio en paralelo, comparar resultados antes de hacer el switch |
| **Feature Flags** | Activar/desactivar el nuevo servicio por usuario, porcentaje de tráfico o entorno |
| **Event Interception** | Interceptar eventos del monolito para alimentar el nuevo servicio durante la transición |

---

## 4. Ejercicio Práctico: Migrando un E-commerce Real / Hands-on Exercise: Migrating a Real E-commerce

### Premisa clave / Key Claim

> **"Simplemente dividir un código grande en un conjunto arbitrario de microservicios no nos dará ningún beneficio."**
>
> **"Just breaking a large codebase into an arbitrary set of microservices won't give us any benefits."**

**Español:**
Este ejercicio existe para demostrar exactamente esa idea. La migración a microservicios solo agrega valor cuando los límites se definen con criterio: siguiendo el negocio, los dominios y los patrones correctos. Hacerlo mal produce un **monolito distribuido** — todo el dolor de los microservicios sin ninguna de sus ventajas.

**English:**
This exercise exists to prove exactly that point. Migrating to microservices only adds value when boundaries are defined thoughtfully: following the business, the domains, and the right patterns. Doing it wrong produces a **distributed monolith** — all the pain of microservices with none of the benefits.

---

### El sistema: ShopFast — E-commerce monolítico exitoso / The System: ShopFast — A Successful Monolithic E-commerce

**Español:**
Imagina que eres el arquitecto de **ShopFast**, una tienda en línea con 5 años en el mercado, miles de clientes activos y un equipo de 40 desarrolladores. El sistema fue construido como un monolito en sus inicios (la decisión correcta en ese momento) y ha funcionado bien. Sin embargo, el crecimiento ha comenzado a crear problemas reales.

**English:**
Imagine you are the architect of **ShopFast**, an online store with 5 years in the market, thousands of active customers, and a team of 40 developers. The system was built as a monolith from the start (the right decision at the time) and has worked well. However, growth has started to create real problems.

---

### Descripción del sistema actual / Current System Description

ShopFast es una arquitectura de **3 capas (3-Tier Architecture)**:

```
╔═══════════════════════════════════════════════════════════════════╗
║                  PRESENTATION TIER (Capa de presentación)        ║
║                                                                   ║
║   ┌──────────────────────┐      ┌──────────────────────┐         ║
║   │      Web App         │      │     Mobile App        │         ║
║   │  (React — Browser)   │      │  (React Native —      │         ║
║   │                      │      │   iOS & Android)      │         ║
║   └──────────┬───────────┘      └──────────┬────────────┘         ║
║              └──────────────┬──────────────┘                      ║
╚═════════════════════════════╪═════════════════════════════════════╝
                              │  HTTP/REST requests
╔═════════════════════════════╪═══════════════════════════════════════════════╗
║              BUSINESS TIER  ↓  (Capa de negocio)                             ║
║                                                                               ║
║  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────────────┐   ║
║  │   Users    │  │  Catalog   │  │   Orders   │  │  Payments Module     │   ║
║  │  Module    │  │  Module    │  │   Module   │  │                      │   ║
║  └────────────┘  └────────────┘  └────────────┘  └──────────┬───────────┘   ║
║  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐              ║
║  │ Inventory  │  │  Shipping  │  │ Promotions │  │ Reviews  │              ║
║  │  Module    │  │  Module    │  │   Module   │  │  Module  │              ║
║  └────────────┘  └────────────┘  └────────────┘  └──────────┘              ║
║  ┌────────────┐  ┌────────────┐                             │               ║
║  │   Search   │  │  Reports   │   ~200,000 líneas de código │               ║
║  │  Module    │  │  Module    │   Single deploy — todo o nada               ║
║  └────────────┘  └────────────┘                             │               ║
╚════════════════════════════════════════════════════════════ ╪ ══════════════╝
                                                              │ HTTPS/REST
                                                   ┌──────────▼───────────┐
                                                   │  External Payment     │
                                                   │     Service           │
                                                   │  (e.g. Stripe /       │
                                                   │        PayPal)        │
                                                   └──────────────────────┘
                              │
╔═════════════════════════════╪═════════════════════════════════════╗
║                DATA TIER    ↓  (Capa de datos)                   ║
║                                                                   ║
║          ┌────────────────────────────────────┐                  ║
║          │       Shared PostgreSQL Database    │                  ║
║          │   54 tables — todos los módulos     │                  ║
║          │   comparten la misma BD             │                  ║
║          │   JOINs cruzados entre módulos      │                  ║
║          └────────────────────────────────────┘                  ║
╚═══════════════════════════════════════════════════════════════════╝
```

> **Nota / Note:** La Presentation Tier ya está **desacoplada** del monolito — consume la API via HTTP. El problema está en el Business Tier y el Data Tier: todo el código de negocio y todos los datos conviven en un único bloque desplegable.

---

### Datos del sistema / System Facts

| Característica / Characteristic | Valor / Value |
|---|---|
| Clientes activos / Active customers | ~80,000 |
| Pedidos diarios / Daily orders | ~5,000 |
| Productos en catálogo / Catalog products | ~120,000 |
| Tamaño del codebase | ~200,000 líneas de código |
| Tiempo de deploy / Deploy time | 45 minutos + 10 min downtime |
| Equipo / Team | 40 desarrolladores en 6 equipos |
| Frecuencia de deploy / Deploy frequency | 1 vez por semana (martes, 2am) |
| Base de datos / Database | 1 PostgreSQL con 54 tablas |
| Tecnología / Technology | Node.js monolito, React frontend |

---

### Puntos de dolor actuales / Current Pain Points

**Español:** Estos son los problemas reales que justifican la migración:
**English:** These are the real problems that justify the migration:

```
❌ Pain Point 1 — Deploys bloqueados:
   El equipo de Catalog quiere hacer 3 deploys por día (mucho cambio de productos).
   El equipo de Payments solo puede hacer deploy los martes a las 2am.
   Resultado: todos esperan a todos. El negocio pierde velocidad.

❌ Pain Point 2 — Escalabilidad ineficiente:
   Los lunes por la mañana, el módulo de Search tiene 10x el tráfico normal.
   Para escalar Search, hay que escalar TODO el monolito.
   Costo mensual en infraestructura: innecesariamente alto.

❌ Pain Point 3 — Base de datos al límite:
   Las queries de Reports hacen JOINs masivos que lentifican las queries
   de Orders en producción. Comparten la misma DB y se afectan mutuamente.

❌ Pain Point 4 — Onboarding lento:
   Un nuevo desarrollador tarda ~3 meses en sentirse productivo porque
   debe entender las 200,000 líneas antes de poder tocar algo sin romper otra cosa.

❌ Pain Point 5 — Bugs en cascada:
   Un bug en el módulo de Promotions tiró el módulo de Orders la semana pasada.
   Un bug en Reports hizo que toda la app respondiera lentamente durante 2 horas.
```

---

### Tu misión / Your Mission

**Español:**
Con base en todo lo aprendido, tu trabajo es diseñar la estrategia de migración de ShopFast. Los próximos apartados de este ejercicio te guiarán a través de:

1. **Identificar los Bounded Contexts** del dominio de ShopFast
2. **Definir los límites** de cada microservicio (¿qué incluye? ¿qué datos posee?)
3. **Ordenar la migración** — ¿qué servicio extraes primero y por qué?
4. **Diseñar la comunicación** entre servicios (sync vs async, eventos)
5. **Planificar la separación de la DB** sin downtime

**English:**
Based on everything learned, your job is to design ShopFast's migration strategy. The next sections of this exercise will guide you through:

1. **Identifying the Bounded Contexts** of the ShopFast domain
2. **Defining the boundaries** of each microservice (what does it include? what data does it own?)
3. **Ordering the migration** — what service do you extract first and why?
4. **Designing communication** between services (sync vs async, events)
5. **Planning the DB separation** without downtime

> **Recuerda / Remember:** El objetivo no es tener la mayor cantidad de microservicios posible. El objetivo es resolver los pain points reales con el menor riesgo posible.

---

## 5. Intento 1 — Dividir por capas de la aplicación / Attempt 1 — Splitting by Application Layers

### La idea / The Idea

**Español:**
Un primer enfoque intuitivo para migrar ShopFast sería aprovechar las **capas lógicas internas** que ya existen en el codebase. La mayoría de aplicaciones monolíticas están organizadas en capas técnicas: una capa de presentación (controllers/routes), una capa de lógica de negocio (services) y una capa de acceso a datos (repositories/models). La idea sería convertir cada capa en un microservicio independiente.

**English:**
A first intuitive approach to migrate ShopFast would be to leverage the **existing logical layers** already present in the codebase. Most monolithic applications are organized in technical layers: a presentation layer (controllers/routes), a business logic layer (services), and a data access layer (repositories/models). The idea would be to turn each layer into an independent microservice.

---

### Cómo se vería / What It Would Look Like

**Español:**
Por ejemplo, podemos tomar la parte del codebase que maneja las solicitudes entrantes de los usuarios, aplica seguridad, valida permisos, y sirve el HTML, JavaScript y CSS al navegador web — y extraer eso como su propio **Store Service**.

**English:**
For example, we can take the part of the codebase that handles incoming user requests, applies security, validates permissions, and serves the HTML, JavaScript and CSS to the web browser — and extract that as its own **Store Service**.

```
Lo que contiene el Store Service (capa de presentación):

  ┌──────────────────────────────────────────────────────┐
  │                   Store Service                      │
  │                                                      │
  │  • Recibe HTTP requests del navegador / mobile app   │
  │  • Autenticación y validación de sesión (JWT/cookies)│
  │  • Autorización y validación de permisos por rol     │
  │  • Sirve HTML, JavaScript y CSS al navegador web     │
  │  • Server-Side Rendering (SSR) de páginas            │
  │  • Rate limiting y protección básica (CSRF, XSS)     │
  │                                                      │
  │  Ejemplos de código que entraría aquí:               │
  │    app.use(authMiddleware)                           │
  │    app.use(permissionsMiddleware)                    │
  │    app.get('/', renderHomePage)                      │
  │    app.use(express.static('./public'))               │
  └──────────────────────────────────────────────────────┘
```

De la misma forma, la capa de lógica de negocio podría convertirse en otro microservicio:

```
Lo que contiene el Business Logic Service (capa de negocio):

  ┌──────────────────────────────────────────────────────┐
  │             Business Logic Service                   │
  │                                                      │
  │  • Toda la lógica de negocio de la aplicación        │
  │  • Reglas de negocio: descuentos, validaciones,      │
  │    cálculo de precios, flujo de pedidos              │
  │  • Orquestación entre módulos (users, orders,        │
  │    products, payments, inventory, shipping...)       │
  │  • Procesamiento de eventos internos                 │
  │                                                      │
  │  Ejemplos de código que entraría aquí:               │
  │    orderService.createOrder(userId, items)           │
  │    paymentService.processPayment(orderId, amount)    │
  │    inventoryService.reserveStock(productId, qty)     │
  │    promoService.applyDiscount(cart, couponCode)      │
  └──────────────────────────────────────────────────────┘
```

Y finalmente, la capa de datos como el tercer microservicio:

```
Lo que contiene el Data Service (capa de datos):

  ┌──────────────────────────────────────────────────────┐
  │                  Data Service                        │
  │                                                      │
  │  • Toda la interacción con la base de datos          │
  │  • Queries SQL / ORM (find, save, update, delete)    │
  │  • Migraciones y esquema de la DB                    │
  │  • Caché (Redis) y búsqueda (Elasticsearch)          │
  │  • Acceso a todas las 54 tablas de ShopFast          │
  │                                                      │
  │  Ejemplos de código que entraría aquí:               │
  │    userRepo.findById(userId)                         │
  │    orderRepo.save(order)                             │
  │    productRepo.findByCategory(categoryId)            │
  │    db.query('SELECT * FROM orders WHERE ...')        │
  └──────────────────────────────────────────────────────┘
```

**ES:** A primera vista parece una separación limpia. Pero veamos los problemas que genera.
**EN:** At first glance it seems like a clean separation. But let's look at the problems it creates.

---

### Estructura completa del monolito dividido por capas / Full Layer-Split Structure

```
Estructura actual del monolito:        Resultado de dividir por capas:
                                       
  src/                                 ┌─────────────────────────┐
    controllers/    ──────────────────▶│   Store Service         │
      users.js                         │  (auth, permisos,       │
      orders.js                        │   HTML/JS/CSS, routing) │
      products.js                      └───────────┬─────────────┘
      payments.js                                  │
                                                   ▼
    services/       ──────────────────▶┌─────────────────────────┐
      userService.js                   │   Business Logic Service│
      orderService.js                  │  (todos los services)   │
      productService.js                └───────────┬─────────────┘
      paymentService.js                            │
                                                   ▼
    repositories/   ──────────────────▶┌─────────────────────────┐
      userRepo.js                      │   Data Service          │
      orderRepo.js                     │  (todos los repos + DB) │
      productRepo.js                   └─────────────────────────┘
```

---

### ¿Por qué NO funciona? / Why Does It NOT Work?

**Español:**
Este enfoque parece razonable pero produce exactamente lo que queremos evitar: un **monolito distribuido**. Veamos por qué:

**English:**
This approach seems reasonable but produces exactly what we want to avoid: a **distributed monolith**. Here's why:

#### ❌ Problema 1 — Acoplamiento total / Total Coupling

**ES:** Los tres servicios están **encadenados sincrónicamente** para cada operación. Para crear un pedido, el API Service llama al Business Logic Service, que llama al Data Service. Si cualquiera de los tres falla, la operación falla.

**EN:** The three services are **synchronously chained** for every operation. To create an order, the API Service calls the Business Logic Service, which calls the Data Service. If any of the three fails, the entire operation fails.

```
Request: POST /orders

API Service → Business Logic Service → Data Service → DB
     ↑                                      ↓
     └──────────────────────────────────────┘
     
Latencia agregada por los saltos de red:
  Monolito:         ~5ms   (llamadas en memoria)
  Dividido por capas: ~35ms  (3 llamadas HTTP + serialización)
  
Sin beneficio de resiliencia. Con mucha más latencia.
```

#### ❌ Problema 2 — Deployments siguen siendo dependientes / Deployments Still Dependent

**ES:** Si cambias la firma de un método en Business Logic Service, debes actualizar API Service al mismo tiempo. Los tres servicios deben desplegarse juntos o se rompen. Nada cambió respecto al monolito original.

**EN:** If you change a method signature in Business Logic Service, you must update API Service simultaneously. All three services must be deployed together or they break. Nothing changed from the original monolith.

#### ❌ Problema 3 — Ningún pain point resuelto / No Pain Point Solved

**ES:** Recordemos los problemas reales de ShopFast:

**EN:** Let's revisit ShopFast's real problems:

| Pain Point | ¿Lo resuelve dividir por capas? / Does splitting by layers fix it? |
|---|---|
| Deploys bloqueados entre equipos | ❌ No — los 3 servicios siguen desplegándose juntos |
| Escalabilidad ineficiente de Search | ❌ No — Search sigue dentro del Business Logic Service con todo lo demás |
| Queries de Reports que lentifican Orders | ❌ No — comparten el mismo Data Service |
| Onboarding lento | ❌ No — ahora hay 3 repos igual de grandes y acoplados |
| Bugs en cascada | ❌ Peor — un bug en Data Service tumba TODO |

---

### Conclusión del Intento 1 / Conclusion of Attempt 1

> **ES:** Dividir por capas técnicas viola el principio más importante: **los microservicios deben organizarse alrededor de capacidades de negocio, no de capas técnicas**. El resultado es un sistema con toda la complejidad operacional de los microservicios (red, serialización, deploys distribuidos) y ninguno de sus beneficios.
>
> **EN:** Splitting by technical layers violates the most important principle: **microservices must be organized around business capabilities, not technical layers**. The result is a system with all the operational complexity of microservices (network, serialization, distributed deployments) and none of the benefits.

```
✅ Lo que queremos lograr:       ❌ Lo que produce el Intento 1:
                                 
  Cada servicio =                  Cada "servicio" =
  una capacidad de negocio         una capa técnica
  completa y autónoma              acoplada a las otras dos
  
  Orders Service:                  API Service:
    - controllers/orders             - controllers/users
    - services/orders                - controllers/orders
    - repositories/orders            - controllers/products
    - orders DB                      - controllers/payments
                                     - ... (todo mezclado)
```

**ES:** En el siguiente intento veremos el enfoque correcto: dividir por **dominio de negocio**.
**EN:** In the next attempt we'll see the correct approach: splitting by **business domain**.

---

## 6. Intento 2 — Dividir por tecnología / Attempt 2 — Splitting by Technology Boundaries

### La idea / The Idea

**Español:**
Otro enfoque común es dividir el monolito según la **tecnología o stack utilizado** por cada parte del sistema. En ShopFast, diferentes módulos podrían beneficiarse de diferentes tecnologías: el motor de búsqueda necesita Elasticsearch, las recomendaciones necesitan Python/ML, los reportes necesitan un motor analítico. La idea es extraer cada parte que use una tecnología diferente como su propio servicio.

**English:**
Another common approach is to split the monolith based on the **technology or stack used** by each part of the system. In ShopFast, different modules could benefit from different technologies: the search engine needs Elasticsearch, recommendations need Python/ML, reports need an analytical engine. The idea is to extract each part that uses a different technology as its own service.

---

### Cómo se vería / What It Would Look Like

```
ShopFast dividido por tecnología:

  ┌──────────────────────────────────────────────────────────────┐
  │              Node.js Service                                 │
  │  Users, Orders, Payments, Inventory, Shipping,              │
  │  Promotions, Reviews  (lógica principal del negocio)        │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │              Python / ML Service                             │
  │  Recommendations Engine, Fraud Detection,                   │
  │  Dynamic Pricing Algorithm                                   │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │              Search Service (Elasticsearch)                  │
  │  Full-text search, filters, facets,                         │
  │  autocomplete, product indexing                             │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │              Reports Service (ClickHouse / Spark)           │
  │  Analytics, dashboards, sales reports,                      │
  │  business intelligence queries                              │
  └──────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │              Real-time Service (WebSockets / Redis)         │
  │  Live notifications, order tracking,                        │
  │  inventory updates in real time                             │
  └──────────────────────────────────────────────────────────────┘
```

---

### ✅ Pros — Dónde tiene sentido / Where It Makes Sense

| Pro | Descripción ES | Description EN |
|---|---|---|
| **Libertad tecnológica** | Cada servicio puede usar el lenguaje y framework más adecuado para su problema | Each service can use the language and framework best suited for its problem |
| **Optimización real** | Search con Elasticsearch es genuinamente más rápido que SQL LIKE. Reports con ClickHouse procesa billones de filas que PostgreSQL no puede | Search with Elasticsearch is genuinely faster than SQL LIKE. Reports with ClickHouse processes billions of rows that PostgreSQL cannot |
| **Aislamiento de carga** | Reports ya no puede saturar la DB principal de Orders con sus queries pesadas | Reports can no longer saturate Orders' main DB with heavy queries |
| **Equipos especializados** | El equipo de Data Science puede trabajar en Python de forma autónoma, sin depender del equipo de Node.js | The Data Science team can work in Python autonomously, without depending on the Node.js team |
| **Separación natural** | Cuando la diferencia tecnológica es tan grande (ML vs CRUD), la separación sí reduce el acoplamiento real | When the technology difference is so large (ML vs CRUD), the split does reduce real coupling |

```
✅ Caso real donde funciona bien:

  Orders Service (Node.js)  ──→  guarda pedido en PostgreSQL
         │
         │  publica evento: "order.created"
         ▼
  Search Service (Elasticsearch)  ←─  indexa productos para búsqueda
  Reports Service (ClickHouse)    ←─  agrega métricas de ventas
  ML Service (Python/TensorFlow)  ←─  actualiza modelo de recomendaciones

  Estos servicios tienen tecnologías TAN diferentes que la separación
  es natural y los equipos son genuinamente independientes.
```

---

### ❌ Contras — Dónde falla / Where It Falls Short

| Contra | Descripción ES | Description EN |
|---|---|---|
| **No resuelve el problema principal** | El Node.js Service sigue siendo el monolito original, solo sin los módulos periféricos. Orders, Payments, Users, Inventory siguen juntos y bloqueando los deploys entre equipos | The Node.js Service is still the original monolith, just without the peripheral modules. Orders, Payments, Users, Inventory are still together and blocking team deployments |
| **Límites forzados por stack, no por negocio** | Si mañana Search se reescribe en Node.js, ¿desaparece el servicio? Los límites deben venir del dominio, no del lenguaje | If Search is rewritten in Node.js tomorrow, does the service disappear? Boundaries must come from the domain, not the language |
| **Consistencia de datos compleja** | Un pedido creado en Node.js debe sincronizarse con Python para recomendaciones, con Elasticsearch para search, con ClickHouse para reports. Múltiples fuentes de verdad | An order created in Node.js must sync to Python for recommendations, Elasticsearch for search, ClickHouse for reports. Multiple sources of truth |
| **Overhead operacional sin beneficio proporcional** | Ahora hay 5 stacks tecnológicos distintos para operar, monitorear y mantener. El equipo necesita conocer Node.js, Python, Elasticsearch, ClickHouse y Redis | Now there are 5 different technology stacks to operate, monitor and maintain. The team needs to know Node.js, Python, Elasticsearch, ClickHouse and Redis |

```
❌ Lo que NO resuelve el Intento 2 en ShopFast:

  Pain Point: El equipo de Catalog quiere hacer 3 deploys/día
              pero el equipo de Payments solo puede el martes.

  Node.js Service (sigue siendo un monolito):
    ┌──────────────────────────────────────────────┐
    │  Users + Orders + Payments + Inventory +     │
    │  Shipping + Promotions + Reviews + Catalog   │
    │                                              │
    │  → Siguen desplegándose JUNTOS               │
    │  → El equipo de Catalog sigue bloqueado      │
    └──────────────────────────────────────────────┘
```

---

### Conclusión del Intento 2 / Conclusion of Attempt 2

> **ES:** Dividir por tecnología tiene valor **cuando la diferencia tecnológica es tan grande que justifica un equipo y ciclo de vida separados** (ML, Analytics, Real-time). Pero no es una estrategia de migración completa. El monolito Node.js principal sigue existiendo. Resolver los pain points reales de ShopFast requiere dividir ese monolito por **dominio de negocio**, que es lo que veremos en el siguiente intento.
>
> **EN:** Splitting by technology has value **when the technology difference is large enough to justify a separate team and lifecycle** (ML, Analytics, Real-time). But it's not a complete migration strategy. The main Node.js monolith still exists. Solving ShopFast's real pain points requires splitting that monolith by **business domain**, which is what we'll see in the next attempt.

| | Intento 1 (capas) | Intento 2 (tecnología) | Intento 3 (tamaño mínimo) | Intento 4 (dominio) |
|---|---|---|---|---|
| Deploys independientes | ❌ | ⚠️ Parcial | ⚠️ Técnicamente sí, pero acoplados | ✅ |
| Escalabilidad granular | ❌ | ⚠️ Parcial | ⚠️ Sí, pero overhead brutal | ✅ |
| Equipos autónomos | ❌ | ⚠️ Solo para ML/Search | ❌ Demasiados servicios por persona | ✅ |
| Aislamiento de fallos | ❌ Peor | ⚠️ Parcial | ❌ Peor — cascada de llamadas | ✅ |
| Complejidad añadida | Alta | Media-Alta | Muy alta sin beneficio | Justificada |

---

## 7. Intento 3 — Dividir por tamaño mínimo / Attempt 3 — Splitting for Minimum Size

### La idea / The Idea

**Español:**
Un tercer enfoque surge de una interpretación literal de los principios de microservicios: *"cada servicio debe hacer una sola cosa y hacerla bien"*, *"servicios pequeños son más fáciles de entender y cambiar"*. Llevado al extremo, esto sugiere crear el **mayor número posible de servicios, cada uno lo más pequeño posible** — un servicio por entidad, por operación, o incluso por endpoint. A este antipatrón se le conoce como **nano-servicios** (nano-services).

**English:**
A third approach emerges from a literal interpretation of microservices principles: *"each service should do one thing and do it well"*, *"small services are easier to understand and change"*. Taken to the extreme, this suggests creating **as many services as possible, each as small as possible** — one service per entity, per operation, or even per endpoint. This anti-pattern is known as **nano-services**.

---

### Cómo se vería / What It Would Look Like

**Español:**
En ShopFast, en lugar de un `Orders Service` que gestione todo el ciclo de vida de un pedido, crearíamos un servicio separado para cada operación:

**English:**
In ShopFast, instead of one `Orders Service` managing the full order lifecycle, we'd create a separate service for each operation:

```
Descomposición de "Orders" en nano-servicios:

  ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
  │  Order-Create       │   │  Order-Read         │   │  Order-Update       │
  │  Service            │   │  Service            │   │  Service            │
  │                     │   │                     │   │                     │
  │  POST /orders       │   │  GET /orders/:id    │   │  PATCH /orders/:id  │
  │  Valida el carrito  │   │  GET /orders?user=  │   │  Actualiza estado   │
  │  Reserva inventario │   │  GET /orders/recent │   │  Maneja cancelación │
  └─────────────────────┘   └─────────────────────┘   └─────────────────────┘

  ┌─────────────────────┐   ┌─────────────────────┐
  │  Order-Delete       │   │  Order-Status       │
  │  Service            │   │  Service            │
  │                     │   │                     │
  │  DELETE /orders/:id │   │  GET /orders/status │
  │  Cancela pedidos    │   │  Webhook de tracking│
  └─────────────────────┘   └─────────────────────┘
```

Aplicando esto a **todos los módulos** de ShopFast:

```
ShopFast nano-services (~60 servicios):

  User-Create-Service          User-Read-Service
  User-Update-Service          User-Delete-Service
  User-Auth-Service            User-Profile-Service

  Product-Create-Service       Product-Read-Service
  Product-Update-Service       Product-Delete-Service
  Product-Search-Service       Product-Image-Service
  Product-Price-Service        Product-Category-Service

  Order-Create-Service         Order-Read-Service
  Order-Update-Service         Order-Cancel-Service
  Order-Status-Service         Order-History-Service

  Payment-Process-Service      Payment-Refund-Service
  Payment-Validate-Service     Payment-History-Service

  Inventory-Reserve-Service    Inventory-Release-Service
  Inventory-Update-Service     Inventory-Check-Service

  Shipping-Create-Service      Shipping-Track-Service
  Shipping-Update-Service      Shipping-Estimate-Service

  ... (y muchos más)
```

---

### ¿Por qué NO funciona? / Why Does It NOT Work?

#### ❌ Problema 1 — Chattiness extrema / Extreme Chattiness

**ES:** Crear un pedido en ShopFast requiere coordinar múltiples pasos. En el monolito, esto se hace en memoria. Con nano-servicios, cada paso es una llamada HTTP a través de la red:

**EN:** Creating an order in ShopFast requires coordinating multiple steps. In the monolith, this is done in-memory. With nano-services, each step becomes an HTTP call across the network:

```
Request: POST /orders  (crear un pedido)

Monolito — todo en memoria:
  orderService.create()
    → userService.validate()       ~0ms (llamada en memoria)
    → productService.getDetails()  ~0ms
    → inventoryService.reserve()   ~0ms
    → paymentService.process()     ~0ms
    → shippingService.estimate()   ~0ms
  Total: ~15ms (solo latencia DB)

Nano-servicios — cada paso es HTTP:
  Order-Create-Service
    → GET  User-Read-Service         ~8ms
    → GET  Product-Read-Service      ~8ms (×N productos en el carrito)
    → POST Inventory-Reserve-Service ~8ms
    → POST Payment-Process-Service   ~8ms
    → GET  Shipping-Estimate-Service ~8ms
    → POST Order-Status-Service      ~8ms
  Total: ~60-150ms (más si hay reintentos)
  
Latencia multiplicada por el número de nano-servicios involucrados.
Cada llamada HTTP agrega serialización, desserialización, conexión TCP y overhead de red.
```

#### ❌ Problema 2 — Transacciones distribuidas imposibles / Impossible Distributed Transactions

**ES:** En el monolito, si el pago falla después de reservar el inventario, se hace rollback en la misma transacción de base de datos. Con nano-servicios, `Inventory-Reserve-Service` y `Payment-Process-Service` tienen bases de datos separadas. Si el pago falla, ¿cómo se libera el inventario reservado? Se necesita un coordinador de transacciones distribuidas (Saga Pattern), que añade una complejidad enorme para un problema que antes no existía.

**EN:** In the monolith, if payment fails after reserving inventory, a rollback happens in the same database transaction. With nano-services, `Inventory-Reserve-Service` and `Payment-Process-Service` have separate databases. If payment fails, how is the reserved inventory released? A distributed transaction coordinator (Saga Pattern) is needed, adding enormous complexity for a problem that didn't exist before.

```
❌ El problema de consistencia en nano-servicios:

  Paso 1: Inventory-Reserve-Service → reserva 1 unidad de producto X  ✅
  Paso 2: Payment-Process-Service   → el pago es rechazado            ❌
  
  ¿Quién libera el inventario reservado en el Paso 1?
  
  Opciones:
    A) Order-Create-Service intenta un rollback manual → frágil, puede fallar
    B) Implementar Saga Pattern → weeks of work, nuevo punto de fallo
    C) Job de compensación periódico → inconsistencia temporal garantizada
    
  En el monolito: db.transaction(() => { reservar(); pagar(); }) → rollback automático
```

#### ❌ Problema 3 — Overhead operacional desproporcionado / Disproportionate Operational Overhead

**ES:** Cada nano-servicio requiere su propio repositorio de código, pipeline de CI/CD, configuración de contenedor, servicio en Kubernetes, reglas de monitoreo, alertas, logs, traces y documentación. Con 60 servicios, un equipo de 40 personas en ShopFast tendría que mantener **más infraestructura que código de negocio**.

**EN:** Each nano-service requires its own code repository, CI/CD pipeline, container configuration, Kubernetes service, monitoring rules, alerts, logs, traces, and documentation. With 60 services, a team of 40 people at ShopFast would end up maintaining **more infrastructure than business code**.

```
Costo operacional por servicio:

  1 nano-servicio = 
    + 1 repositorio Git
    + 1 Dockerfile
    + 1 Kubernetes Deployment + Service + HPA
    + 1 pipeline CI/CD (build → test → deploy)
    + 1 set de reglas en Prometheus/Grafana
    + 1 configuración de alertas en PagerDuty
    + 1 entrada en el service mesh (Istio/Linkerd)
    + 1 set de endpoints en el API Gateway

  60 nano-servicios × todo lo anterior =
    overhead operacional mayor que el monolito original
    
  Un equipo de 40 personas terminará siendo un equipo de operaciones,
  no un equipo de producto.
```

#### ❌ Problema 4 — Límites sin semántica de negocio / Boundaries Without Business Semantics

**ES:** `Order-Create-Service` y `Order-Update-Service` tienen que compartir datos sobre los pedidos. ¿Quién es el dueño de la tabla `orders`? Si ambos acceden a la misma base de datos, es el antipatrón de base de datos compartida. Si tienen bases de datos separadas, ¿cómo `Order-Read-Service` ve los pedidos creados por `Order-Create-Service` en tiempo real? Los límites definidos por operación CRUD no tienen semántica de negocio y generan estas contradicciones irresolubles.

**EN:** `Order-Create-Service` and `Order-Update-Service` both need to share data about orders. Who owns the `orders` table? If both access the same database, it's the shared database anti-pattern. If they have separate databases, how does `Order-Read-Service` see orders created by `Order-Create-Service` in real time? Boundaries defined by CRUD operation have no business semantics and create these unresolvable contradictions.

```
❌ Ambigüedad de propiedad de datos:

  ¿Quién es el dueño de la tabla `orders`?

  Order-Create-Service  → INSERT INTO orders ...
  Order-Update-Service  → UPDATE orders SET status = ...
  Order-Read-Service    → SELECT * FROM orders WHERE ...
  Order-Cancel-Service  → UPDATE orders SET cancelled = true ...
  Order-Status-Service  → SELECT status FROM orders WHERE ...

  Si comparten la misma tabla → Shared Database anti-pattern
                               (ningún servicio tiene autonomía real)
  
  Si tienen bases de datos separadas → ¿cómo se sincronizan?
                                       Consistencia eventual entre operaciones
                                       del mismo "dominio"... absurdo.
```

---

### Conclusión del Intento 3 / Conclusion of Attempt 3

> **ES:** El tamaño mínimo no es un criterio válido para definir límites de microservicios. Un servicio debe ser tan pequeño como necesite ser para encapsular una **capacidad de negocio cohesiva y autónoma**, no tan pequeño como sea técnicamente posible. Los nano-servicios producen un sistema con latencia multiplicada, transacciones distribuidas imposibles, overhead operacional desproporcionado y límites que carecen de semántica real. La granularidad correcta la define el negocio, no el deseo de tener más servicios.
>
> **EN:** Minimum size is not a valid criterion for defining microservice boundaries. A service should be as small as it needs to be to encapsulate a **cohesive and autonomous business capability**, not as small as technically possible. Nano-services produce a system with multiplied latency, impossible distributed transactions, disproportionate operational overhead, and boundaries that lack real semantics. The correct granularity is defined by the business, not by the desire to have more services.

```
La regla que nos falta / The rule we're missing:

  ❌ "Hazlo tan pequeño como puedas"    → nano-servicios
  ❌ "Un servicio por capa técnica"     → Intento 1
  ❌ "Un servicio por tecnología"       → Intento 2 (incompleto)
  
  ✅ "Un servicio por Bounded Context"  → Intento 4
     (lo suficientemente pequeño para ser autónomo,
      lo suficientemente grande para ser coherente)
```

**ES:** En el siguiente intento veremos el enfoque correcto: dividir por **dominio de negocio** usando Bounded Contexts de DDD, que es la estrategia que sí resuelve los pain points reales de ShopFast.
**EN:** In the next attempt we'll see the correct approach: splitting by **business domain** using DDD Bounded Contexts, which is the strategy that actually solves ShopFast's real pain points.

