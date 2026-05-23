# El Principio DRY en Microservicios y Librerías Compartidas / The DRY Principle in Microservices and Shared Libraries

---

## 1. Importancia del principio DRY / Importance of DRY

**Español:**
El principio **DRY** (*Don't Repeat Yourself*) establece que cada pieza de conocimiento o lógica debe tener una única representación autorizada en el sistema. En un monolito esto es directo: si dos módulos necesitan la misma lógica de validación, se extrae a una clase o función compartida. Sin DRY, los bugs se duplican, los cambios deben aplicarse en múltiples lugares y el código se vuelve difícil de mantener.

**English:**
The **DRY** (*Don't Repeat Yourself*) principle states that every piece of knowledge or logic should have a single authoritative representation in the system. In a monolith this is straightforward: if two modules need the same validation logic, it's extracted to a shared class or function. Without DRY, bugs are duplicated, changes must be applied in multiple places, and the code becomes hard to maintain.

```
DRY en un monolito / DRY in a monolith:

  Sin DRY / Without DRY:               Con DRY / With DRY:
  
  OrderModule                           OrderModule
    validateEmail(email) { ... }    ──▶   validateEmail() ──┐
                                                             │
  UserModule                            UserModule           │  shared/
    validateEmail(email) { ... }    ──▶   validateEmail() ──┼─▶ validators.js
                                                             │  validateEmail()
  NotificationModule                    NotificationModule   │
    validateEmail(email) { ... }    ──▶   validateEmail() ──┘

  3 copias → 3 lugares para          1 copia → 1 lugar para
  corregir un bug                    corregir un bug
```

> **ES:** DRY es un principio sólido en arquitecturas monolíticas. El problema surge cuando se intenta aplicarlo tal cual en microservicios — la solución obvia (librería compartida) introduce un nuevo problema: **acoplamiento entre servicios**.
>
> **EN:** DRY is a solid principle in monolithic architectures. The problem arises when trying to apply it directly to microservices — the obvious solution (shared library) introduces a new problem: **coupling between services**.

---

## 2. Desafíos del DRY y las librerías compartidas / Challenges of DRY and Shared Libraries


**Español:**
La solución intuitiva para aplicar DRY en microservicios es crear una **librería compartida** que contenga los modelos, validaciones o utilidades comunes. Sin embargo, esta solución convierte la librería en un punto de acoplamiento fuerte entre todos los servicios que la usan.

**English:**
The intuitive solution to apply DRY in microservices is to create a **shared library** containing common models, validations, or utilities. However, this solution turns the library into a strong coupling point between all services that use it.

```
El problema de las librerías compartidas / The shared library problem:

  shared-models v1.0:
  ┌────────────────────────────────┐
  │  class Order { id, items,      │
  │    customerId, status }        │
  │  class Product { id, name,     │
  │    price, categoryId }         │
  └────────────────────────────────┘
         │        │        │        │
         ▼        ▼        ▼        ▼
  ┌──────────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │  Order   │ │Catal.│ │Inven.│ │Ship. │
  │  Svc     │ │ Svc  │ │ Svc  │ │ Svc  │
  │  v1.0 ✅ │ │ v1.0 │ │ v1.0 │ │ v1.0 │
  └──────────┘ └──────┘ └──────┘ └──────┘

  Escenario: Order Service necesita añadir campo "discountCode" a Order

  shared-models v1.1:
  ┌───────────────────────────────────────┐
  │  class Order { id, items,             │
  │    customerId, status, discountCode } │ ← cambio
  └───────────────────────────────────────┘
         │        │        │        │
         ▼        ▼        ▼        ▼
  ┌──────────┐ ┌──────┐ ┌──────┐ ┌──────┐
  │  Order   │ │Catal.│ │Inven.│ │Ship. │
  │  Svc     │ │ Svc  │ │ Svc  │ │ Svc  │
  │  v1.1 ✅ │ │ v1.1 │ │ v1.1 │ │ v1.1 │
  │          │ │ ⚠️   │ │ ⚠️   │ │ ⚠️   │
  └──────────┘ └──────┘ └──────┘ └──────┘
               deben actualizar y redesplegar aunque
               no usen discountCode para nada
```

**Español:**
Este es el problema central: **una librería compartida de modelos de negocio convierte un cambio de un servicio en un evento de coordinación de toda la organización**. Los servicios pierden su capacidad de desplegarse de forma independiente.

**English:**
This is the core problem: **a shared business-model library turns one service's change into an organization-wide coordination event**. Services lose their ability to deploy independently.

```
Consecuencias concretas / Concrete consequences:

  ✗ Acoplamiento fuerte (tight coupling)
     → Los servicios quedan atados al ciclo de vida de la librería
     → La independencia que prometen los microservicios desaparece

  ✗ Cada cambio exige: Rebuild → Retest → Redeploy de todos los consumidores
     → Un fix de un campo en Order obliga a:
       1. Publicar nueva versión de la librería
       2. Cada equipo actualiza su dependencia
       3. Cada servicio hace rebuild completo
       4. Cada servicio pasa sus test suites
       5. Cada servicio se redespliega a producción
     → Lo que debía ser un cambio de 1 servicio se convierte en N deploys

  ✗ Bug o vulnerabilidad en la librería impacta TODOS los microservicios
     → Un fallo de seguridad (ej. CVE en una dependencia transitiva) expone
       simultáneamente todos los servicios que usan la librería
     → Obliga a un parche de emergencia coordinado en toda la organización
     → Cuantos más servicios usen la librería, mayor el radio de blast

  ✗ Versioning complejo: ¿qué servicio usa v1.0, v1.1, v1.2?
  ✗ "Dependency hell": conflictos entre versiones transitivas de la librería
     → Service A usa shared-lib v1.2 que requiere lodash v4
     → Service B usa shared-lib v1.0 que requiere lodash v3
     → Ambos servicios comparten el mismo runtime → conflicto irresolvible
     → Actualizar la librería para un servicio puede romper silenciosamente otro
  ✗ El monorepo de la librería se convierte en un cuello de botella del equipo
  ✗ Regresiones inesperadas: un cambio para Order rompe Catalog sin querer
  ✗ Resultado: un "monolito distribuido" — peor que el monolito original
```

> **ES:** Pogrebinsky advierte que una librería compartida de **lógica de negocio o modelos de dominio** es uno de los anti-patrones más comunes al migrar a microservicios. La clave es distinguir qué tipo de código es legítimo compartir y qué no.
>
> **EN:** Pogrebinsky warns that a shared library of **business logic or domain models** is one of the most common anti-patterns when migrating to microservices. The key is distinguishing what type of code is legitimate to share and what is not.

---

## 3. Alternativas a las librerías compartidas / Alternatives to Shared Libraries

**Español:**
La respuesta al dilema DRY vs. autonomía no es "nunca compartir nada", sino **ser selectivo sobre qué se comparte**. Pogrebinsky propone una distinción clara: compartir está permitido solo para código de **infraestructura**, nunca para código de **dominio de negocio**.

**English:**
The answer to the DRY vs. autonomy dilemma is not "never share anything", but **being selective about what is shared**. Pogrebinsky proposes a clear distinction: sharing is allowed only for **infrastructure** code, never for **business domain** code.

```
Qué está permitido compartir / What is OK to share:

  ✅ Librerías de infraestructura (NO lógica de negocio):
  ┌─────────────────────────────────────────────────────┐
  │  • Logging estructurado (formato JSON unificado)     │
  │  • HTTP client con retry y circuit breaker           │
  │  • Health check / readiness endpoints estándar       │
  │  • Distributed tracing (instrumentación OpenTelemetry│
  │  • Autenticación JWT (parsing y validación del token)│
  │  • Configuración de entornos (env vars)              │
  └─────────────────────────────────────────────────────┘

  ❌ Qué NO compartir / What NOT to share:
  ┌─────────────────────────────────────────────────────┐
  │  • Modelos de dominio (Order, Product, Customer...)  │
  │  • Reglas de negocio (cálculo de descuentos, etc.)   │
  │  • Esquemas de base de datos                         │
  │  • DTOs o contratos de API entre servicios           │
  └─────────────────────────────────────────────────────┘
```

### Alternativa 1: Duplicación intencional / Intentional Duplication

**Español:**
Cuando dos servicios necesitan una estructura de datos similar (por ejemplo, ambos necesitan una representación de "Product"), la solución correcta es que **cada servicio tenga su propia copia**, adaptada a sus necesidades. Esto rompe DRY intencionalmente a cambio de autonomía total.

**English:**
When two services need a similar data structure (for example, both need a representation of "Product"), the correct solution is for **each service to have its own copy**, adapted to its needs. This intentionally breaks DRY in exchange for full autonomy.

```
Duplicación intencional en ShopFast / Intentional duplication in ShopFast:

  Catalog Service:                    Order Service:
  ┌────────────────────────┐          ┌────────────────────────┐
  │ Product {               │          │ OrderItem {             │
  │   id, name, description,│          │   productId,            │
  │   price, categoryId,    │          │   productName,  ← copia │
  │   images[], tags[],     │          │   priceAtOrder, ← del   │
  │   weight, dimensions    │          │   quantity      ← momento│
  │ }                       │          │ }               ← exacto │
  └────────────────────────┘          └────────────────────────┘

  → Catalog tiene el modelo completo de producto para gestión
  → Order solo guarda lo que necesita en el momento del pedido
  → Son modelos distintos aunque compartan algunos campos
```

### Alternativa 2: Exposición vía API / Expose via API

**Español:**
En lugar de compartir código, compartir **comportamiento a través de una API**. Si la lógica de validación de emails es compleja y crítica, se crea un servicio de validación que expone un endpoint. Los otros servicios lo llaman vía HTTP en lugar de importar una librería.

**English:**
Instead of sharing code, share **behavior through an API**. If email validation logic is complex and critical, create a validation service that exposes an endpoint. Other services call it via HTTP instead of importing a library.

```
Librería compartida vs API de servicio / Shared library vs Service API:

  ❌ Shared library:              ✅ Service API:
  
  import { validate }             POST /api/validate/email
    from 'shared-validators'        body: { email: "..." }
                                    response: { valid: true }
  
  → acoplamiento en build time    → acoplamiento solo en runtime
  → redesploy coordinado          → cada servicio se despliega solo
  → versioning de paquete         → versionado de API (v1, v2)
```

### Alternativa 3: Copiar y divergir / Copy and Diverge

**Español:**
Para utilidades pequeñas (una función de formato de fecha, un helper de paginación), la solución más pragmática es simplemente **copiarla en cada servicio que la necesite** y aceptar que con el tiempo divergirán. El costo de la duplicación es menor que el costo del acoplamiento.

**English:**
For small utilities (a date formatting function, a pagination helper), the most pragmatic solution is to simply **copy it into each service that needs it** and accept that over time they will diverge. The cost of duplication is lower than the cost of coupling.

### Alternativa 4: Patrón Sidecar / Sidecar Pattern

**Español:**
El **patrón Sidecar** despliega la funcionalidad compartida como un **proceso separado** que corre junto a cada microservicio en el mismo host o pod, en lugar de importarla como una librería. Cada servicio tiene su propio sidecar independiente — si se actualiza el sidecar, solo afecta a ese servicio, no a todos.

**English:**
The **Sidecar pattern** deploys shared functionality as a **separate process** running alongside each microservice on the same host or pod, instead of importing it as a library. Each service has its own independent sidecar — if the sidecar is updated, it only affects that service, not all of them.

```
Sidecar Pattern — estructura / structure:

  Pod / Host A:                        Pod / Host B:
  ┌────────────────────────────┐       ┌────────────────────────────┐
  │  ┌──────────────┐          │       │  ┌──────────────┐          │
  │  │ Order Service│          │       │  │Catalog Service│         │
  │  │   (app)      │◀─────────┼──┐    │  │   (app)      │◀────────┼──┐
  │  └──────────────┘   local  │  │    │  └──────────────┘  local  │  │
  │         │           comm   │  │    │         │          comm   │  │
  │         ▼                  │  │    │         ▼                  │  │
  │  ┌──────────────┐          │  │    │  ┌──────────────┐          │  │
  │  │   Sidecar    │          │  │    │  │   Sidecar    │          │  │
  │  │ (logging,    │──────────┼──┘    │  │ (logging,    │──────────┼──┘
  │  │  tracing,    │          │       │  │  tracing,    │          │
  │  │  auth, TLS)  │          │       │  │  auth, TLS)  │          │
  │  └──────────────┘          │       │  └──────────────┘          │
  └────────────────────────────┘       └────────────────────────────┘

  → Cada sidecar es independiente: actualizar uno no afecta al otro
  → El servicio principal no sabe que el sidecar existe — transparente
  → Ejemplo real: Envoy Proxy en service meshes (Istio, Linkerd)
```

**Español:**
Casos de uso ideales para el Sidecar: logging y tracing transparentes, mTLS entre servicios, circuit breaking, métricas de red. Todo aquello que es **infraestructura transversal** pero que no debe acoplarse al código del servicio.

**English:**
Ideal use cases for Sidecar: transparent logging and tracing, mTLS between services, circuit breaking, network metrics. Everything that is **cross-cutting infrastructure** but must not couple to the service's code.

> **ES:** La regla de Pogrebinsky: **"Un poco de duplicación es mejor que un poco de dependencia incorrecta."** Si compartir algo obliga a coordinar despliegues, no vale la pena compartirlo.
>
> **EN:** Pogrebinsky's rule: **"A little duplication is better than a little wrong dependency."** If sharing something forces deployment coordination, it's not worth sharing.

---

## 4. Compartir / Duplicar datos entre microservicios / Sharing / Duplicating Data Across Microservices

**Español:**
Más allá del código, el mismo dilema existe para los **datos**: ¿debe cada servicio consultar al servicio propietario cada vez que necesita un dato, o debe mantener su propia copia local? Ambos enfoques tienen beneficios y costos concretos.

**English:**
Beyond code, the same dilemma exists for **data**: should each service query the owning service every time it needs a piece of data, or should it maintain its own local copy? Both approaches have concrete benefits and costs.

```
Reglas fundamentales al duplicar datos / Fundamental rules when duplicating data:

  1. Un solo dueño / source of truth (Only one owner)
     → Solo UN servicio puede escribir y modificar un dato
     → Los demás servicios que tienen copias son consumidores de solo lectura
     → Ejemplo: el precio de un producto SOLO lo escribe Catalog Service
       Order Service tiene una copia, pero NUNCA la modifica directamente

     ✅ Catalog Service  →  escribe precio  →  PostgreSQL (catalog DB)
                        →  publica evento ProductPriceUpdated
     ✅ Order Service    →  recibe evento   →  actualiza su copia local
                        →  NUNCA hace UPDATE directo a datos de Catalog

     Si dos servicios pueden escribir el mismo dato → conflicto de escritura
     → datos corruptos → imposible saber cuál es la versión correcta

  2. Solo podemos garantizar consistencia eventual / Only eventual consistency
     → Con datos distribuidos entre servicios, NO es posible garantizar
       que todas las copias estén sincronizadas en el mismo instante
     → Siempre habrá una ventana de tiempo donde las copias divergen

     Tiempo 0: Catalog actualiza precio de "Laptop X1" a $999
     Tiempo 0: Order Service aún tiene el precio anterior ($1099)
     Tiempo 1ms - 500ms: el evento viaja por el Message Bus
     Tiempo ~500ms: Order Service recibe el evento y actualiza su copia
     → Durante esos ~500ms, Order mostraba un precio incorrecto

     → Esto es aceptable para la mayoría de casos de negocio
     → NO es aceptable para datos financieros críticos (ej. saldo de cuenta)
       → esos casos deben consultar al propietario en tiempo real
```

```
Dos estrategias / Two strategies:

  Estrategia A: consultar al servicio propietario / Query the owning service
  
  Order Service ──────── GET /api/products/:id ──────▶ Catalog Service
                ◀─────── { name, price, ... } ─────────

  Estrategia B: replicar datos localmente / Replicate data locally
  
  Catalog Service ──publica evento ProductUpdated──▶ Message Bus
                                                           │
                                           Order Service ◀─┘
                                           guarda copia local en su DB
```

### Beneficios de duplicar / replicar datos / Benefits of Duplicating / Replicating Data

```
✅ Beneficios de la replicación local:

  1. Autonomía y resiliencia
     → Order Service funciona aunque Catalog Service esté caído
     → No hay dependencia de disponibilidad entre servicios

  2. Latencia cero en lecturas locales
     → Sin round-trip de red para obtener el nombre del producto
     → Especialmente importante en servicios con alto volumen de lecturas

  3. Datos congelados en el momento correcto
     → El precio del producto en un pedido debe ser el precio del momento
       de la compra, no el precio actual
     → La copia local garantiza esto de forma natural

  4. Queries complejas sin JOINs entre servicios
     → Order Service puede hacer queries que combinen datos de pedidos
       y datos de productos sin llamadas externas
```

### Desventajas de duplicar / replicar datos / Downsides of Duplicating / Replicating Data

```
❌ Desventajas de la replicación local:

  1. Consistencia eventual (Eventual Consistency)
     → Si Catalog actualiza el precio de un producto, Order Service
       puede tener el precio viejo durante un tiempo
     → Hay una ventana de inconsistencia mientras el evento se propaga

  2. Complejidad de sincronización
     → Se necesita un mecanismo de eventos (Kafka, RabbitMQ) para
       mantener las copias actualizadas
     → Si se pierde un evento, las copias quedan desincronizadas

  3. Duplicación de almacenamiento
     → Los mismos datos (nombres de productos) existen en N bases de datos
     → Mayor uso de disco y costos de infraestructura

  4. Riesgo de datos obsoletos (Stale Data)
     → Si la sincronización falla silenciosamente, el servicio trabaja
       con datos incorrectos sin saberlo
     → Requiere mecanismos de reconciliación periódica
```

### Tabla comparativa / Comparison Table

| Criterio | Consultar al propietario | Replicar localmente |
|---|---|---|
| Consistencia | ✅ Siempre datos frescos | ❌ Eventual consistency |
| Disponibilidad | ❌ Depende del servicio propietario | ✅ Autónomo |
| Latencia | ❌ Round-trip de red | ✅ Lectura local |
| Complejidad | ✅ Simple (HTTP call) | ❌ Requiere sistema de eventos |
| Resiliencia a fallos | ❌ Falla si el propietario cae | ✅ Opera de forma independiente |
| Almacenamiento | ✅ Sin duplicación | ❌ Datos duplicados |

> **ES:** No existe una regla universal. Pogrebinsky enseña a elegir según el caso: si el dato cambia con frecuencia y la consistencia es crítica → consultar al propietario. Si el dato es relativamente estable y la disponibilidad/latencia son prioritarias → replicar localmente con sincronización por eventos.
>
> **EN:** There is no universal rule. Pogrebinsky teaches to choose based on the case: if the data changes frequently and consistency is critical → query the owner. If the data is relatively stable and availability/latency are the priority → replicate locally with event-based synchronization.
