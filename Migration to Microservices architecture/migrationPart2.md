## 1. Descomposición por capacidades de negocio / Decomposition by Business Capabilities
Run a thought experiment:
- Describe the system to a non-technical person.
- explain what the system does / what value each capability provides.

### Definición / Definition

**Español:**
La **descomposición por capacidades de negocio** consiste en identificar las **cosas que el negocio hace** — sus capacidades estables y permanentes — y crear un microservicio por cada una. Una capacidad de negocio es una función que la organización realiza para generar valor: procesar pagos, gestionar pedidos, notificar a clientes. A diferencia de los módulos técnicos, las capacidades de negocio cambian muy lentamente y son independientes de la tecnología usada para implementarlas.

**English:**
**Decomposition by business capabilities** consists of identifying the **things the business does** — its stable, permanent capabilities — and creating one microservice per capability. A business capability is a function the organization performs to generate value: processing payments, managing orders, notifying customers. Unlike technical modules, business capabilities change very slowly and are independent of the technology used to implement them.

> **Regla práctica / Practical rule:** Si puedes describir la capacidad con un sustantivo de negocio ("Payments", "Notifications", "Inventory"), probablemente es un buen servicio. Si necesitas un verbo técnico ("DatabaseAccess", "HTTPHandler"), no lo es.
>
> **Practical rule:** If you can describe the capability with a business noun ("Payments", "Notifications", "Inventory"), it's probably a good service. If you need a technical verb ("DatabaseAccess", "HTTPHandler"), it isn't.

---

### Cómo identificar capacidades de negocio / How to Identify Business Capabilities

**Español:**
El proceso de identificación parte de preguntas de negocio, no de preguntas técnicas:

**English:**
The identification process starts from business questions, not technical ones:

```
Preguntas guía / Guiding questions:

  1. ¿Qué hace este negocio para generar valor?
     What does this business do to generate value?

  2. ¿Qué haría un empleado dedicado exclusivamente a esta área?
     What would an employee dedicated exclusively to this area do?

  3. ¿Qué procesos seguirían funcionando si toda la tecnología se apagara?
     Which processes would keep working if all technology were turned off?

  4. ¿Qué área del negocio tiene su propio presupuesto, KPIs y equipo?
     Which area of the business has its own budget, KPIs and team?
```

---

### Ejemplo en ShopFast / Example in ShopFast

**Español:**
Aplicando este análisis a ShopFast, identificamos las capacidades de negocio principales:

**English:**
Applying this analysis to ShopFast, we identify the core business capabilities:

```
Capacidades de negocio de ShopFast:

  ┌─────────────────────────────────────────────────────────────────────┐
  │                        ShopFast                                     │
  ├──────────────────┬──────────────────┬──────────────────────────────-┤
  │  User Management │  Catalog Mgmt    │  Order Management             │
  │                  │                  │                               │
  │  • Registro      │  • Productos     │  • Crear pedido               │
  │  • Autenticación │  • Categorías    │  • Estado del pedido          │
  │  • Perfiles      │  • Precios       │  • Historial                  │
  │  • Roles         │  • Imágenes      │  • Cancelación                │
  ├──────────────────┼──────────────────┼───────────────────────────────┤
  │  Payment         │  Inventory       │  Shipping                     │
  │  Processing      │  Management      │  & Logistics                  │
  │                  │                  │                               │
  │  • Cobro         │  • Stock actual  │  • Estimación de envío        │
  │  • Reembolsos    │  • Reservas      │  • Tracking                   │
  │  • Facturación   │  • Reposición    │  • Gestión de carriers        │
  │  • Historial     │  • Alertas       │  • Devoluciones               │
  ├──────────────────┼──────────────────┼───────────────────────────────┤
  │  Notifications   │  Search &        │  Promotions &                 │
  │                  │  Discovery       │  Marketing                    │
  │  • Email         │                  │                               │
  │  • SMS           │  • Full-text     │  • Cupones                    │
  │  • Push          │  • Filtros       │  • Descuentos                 │
  │  • In-app        │  • Autocomplete  │  • Campañas                   │
  └──────────────────┴──────────────────┴───────────────────────────────┘

Resultado: 8 microservicios alineados con 8 capacidades de negocio reales.
Result: 8 microservices aligned with 8 real business capabilities.
```

Cada capacidad se convierte en un microservicio con sus propios datos:

```
User Service          → tabla: users, sessions, roles
Catalog Service       → tabla: products, categories, prices, images
Order Service         → tabla: orders, order_items, order_status_history
Payment Service       → tabla: payments, invoices, refunds
Inventory Service     → tabla: inventory, reservations, restock_alerts
Shipping Service      → tabla: shipments, carriers, tracking_events
Notification Service  → tabla: notification_templates, notification_log
Search Service        → índice: Elasticsearch (products index)
Promotions Service    → tabla: coupons, campaigns, discount_rules
```

---

### ✅ Pros / Advantages

| Pro | Descripción ES | Description EN |
|---|---|---|
| **Estabilidad de los límites** | Las capacidades de negocio cambian muy poco con el tiempo. Un servicio definido así no necesita refactorizarse cuando cambia la tecnología | Business capabilities change very little over time. A service defined this way doesn't need refactoring when technology changes |
| **Alineación con equipos** | Un equipo de negocio (ej. equipo de Payments) tiene un servicio correspondiente. Conway's Law trabaja a favor | A business team (e.g., Payments team) has a corresponding service. Conway's Law works in your favor |
| **Vocabulario compartido** | Los ingenieros y el negocio hablan del mismo servicio con el mismo nombre. Reduce malentendidos | Engineers and the business talk about the same service by the same name. Reduces misunderstandings |
| **Autonomía real** | Cada servicio puede desplegarse, escalarse y evolucionar de forma independiente porque sus responsabilidades no se solapan | Each service can be deployed, scaled, and evolved independently because its responsibilities don't overlap |
| **Fácil de razonar** | Cuando algo falla, es claro qué servicio es responsable. Debugging y observabilidad son más directos | When something fails, it's clear which service is responsible. Debugging and observability are more straightforward |

---

### ❌ Contras / Disadvantages

| Contra | Descripción ES | Description EN |
|---|---|---|
| **Requiere conocimiento profundo del negocio** | No se puede aplicar sin entender cómo funciona el negocio. Un ingeniero que no conoce el dominio hará malos cortes | Cannot be applied without understanding how the business works. An engineer unfamiliar with the domain will make poor cuts |
| **Granularidad subjetiva** | ¿"Promotions" y "Pricing" son una capacidad o dos? La respuesta depende del contexto y puede generar debate prolongado | Are "Promotions" and "Pricing" one capability or two? The answer depends on context and can generate prolonged debate |
| **Posibles servicios grandes** | Algunas capacidades son intrínsecamente complejas (ej. Orders). El servicio resultante puede ser grande, lo que puede requerir sub-descomposición posterior | Some capabilities are intrinsically complex (e.g., Orders). The resulting service can be large, potentially requiring later sub-decomposition |
| **Cambios de negocio remueven fronteras** | Si el negocio pivota (ej. fusión de Promotions con Catalog), los límites del servicio quedan mal definidos hasta que se refactorizan | If the business pivots (e.g., merging Promotions with Catalog), service boundaries become poorly defined until refactored |

---

### Conclusión / Conclusion

> **ES:** La descomposición por capacidades de negocio es el punto de partida más robusto para la mayoría de organizaciones. Produce servicios estables, autónomos y comprensibles porque sus límites vienen del negocio real, no de decisiones técnicas. Es especialmente poderosa cuando los equipos de ingeniería están organizados alrededor de áreas de negocio (squads, product teams).
>
> **EN:** Decomposition by business capabilities is the most robust starting point for most organizations. It produces stable, autonomous, and understandable services because its boundaries come from the real business, not from technical decisions. It is especially powerful when engineering teams are organized around business areas (squads, product teams).

---

## 2. Descomposición por dominio / subdominio (DDD) / Decomposition by Domain / Subdomain (DDD)

### Definición / Definition

**Español:**
La **descomposición por dominio** usa **Domain-Driven Design (DDD)** para identificar los límites de cada servicio. DDD parte de modelar el **dominio** (el espacio del problema del negocio) de forma rigurosa, identificando **subdominios** y **Bounded Contexts**. Un **Bounded Context** es una región del dominio donde un modelo, un lenguaje y unas reglas específicas aplican de forma consistente y sin ambigüedad. Cada microservicio debería corresponder (idealmente) a un Bounded Context.

La diferencia clave con las capacidades de negocio es la profundidad: DDD no solo pregunta *"¿qué hace el negocio?"* sino *"¿cómo modela el negocio este concepto, con qué lenguaje y qué reglas?"*

**English:**
**Decomposition by domain** uses **Domain-Driven Design (DDD)** to identify service boundaries. DDD starts by rigorously modeling the **domain** (the business problem space), identifying **subdomains** and **Bounded Contexts**. A **Bounded Context** is a region of the domain where a specific model, language, and rules apply consistently and without ambiguity. Each microservice should (ideally) correspond to one Bounded Context.

The key difference from business capabilities is depth: DDD doesn't just ask *"what does the business do?"* but *"how does the business model this concept, with what language and what rules?"*

---

### Conceptos clave de DDD / Key DDD Concepts

#### Bounded Context

**ES:** Un Bounded Context define el **alcance** dentro del cual un modelo de dominio es válido. El mismo término puede significar cosas distintas en contextos diferentes. Por ejemplo, "Product" en el contexto de Catalog significa algo diferente que "Product" en el contexto de Orders.

**EN:** A Bounded Context defines the **scope** within which a domain model is valid. The same term can mean different things in different contexts. For example, "Product" in the Catalog context means something different than "Product" in the Orders context.

```
Ejemplo de "Product" en distintos Bounded Contexts:

  Catalog Context:              Orders Context:
  ┌──────────────────────┐      ┌──────────────────────┐
  │ Product              │      │ OrderItem (Product)   │
  │ ─────────────────    │      │ ──────────────────── │
  │ id                   │      │ productId (solo ref)  │
  │ name                 │      │ productName (snapshot)│
  │ description          │      │ unitPrice (fijo al    │
  │ images[]             │      │   momento del pedido) │
  │ categories[]         │      │ quantity              │
  │ currentPrice         │      │                       │
  │ stockLevel           │      │ ← No tiene imágenes,  │
  │ attributes{}         │      │   ni descripción,     │
  │                      │      │   ni stock            │
  └──────────────────────┘      └──────────────────────┘

  Son el mismo concepto en el mundo real, pero modelos distintos
  en cada Bounded Context — y eso es correcto y esperado.
```

#### Tipos de subdominio / Types of Subdomain

```
Core Domain (Dominio central):
  → Lo que diferencia al negocio de la competencia
  → Máxima inversión de ingeniería, equipo senior
  → Ejemplo ShopFast: Recommendation Engine, Dynamic Pricing

Supporting Domain (Dominio de soporte):
  → Apoya al Core Domain pero no es la ventaja competitiva
  → Puede desarrollarse internamente pero con menos prioridad
  → Ejemplo ShopFast: Order Management, Inventory

Generic Domain (Dominio genérico):
  → Funcionalidad estándar que cualquier empresa necesita
  → Mejor opción: usar un SaaS o librería existente
  → Ejemplo ShopFast: Authentication → Auth0/Keycloak
                       Email         → SendGrid/AWS SES
                       Payments      → Stripe/PayPal

Subdomain categorization helps
  → Priorizar la inversión en cada subdominio
  → Asignar ingenieros según su nivel de experiencia  
```

---

### Ejemplo en ShopFast / Example in ShopFast

**Español:**
Aplicando DDD a ShopFast, el análisis va más allá de listar capacidades: define el **lenguaje ubicuo** (Ubiquitous Language) de cada contexto y los modelos de dominio específicos.

**English:**
Applying DDD to ShopFast, the analysis goes beyond listing capabilities: it defines the **Ubiquitous Language** of each context and the specific domain models.

```
Bounded Contexts de ShopFast:

┌──────────────────────────────────────────────────────────────────────┐
│  CORE DOMAIN                                                         │
│                                                                      │
│  ┌───────────────────────────┐  ┌───────────────────────────────┐   │
│  │  Catalog Context          │  │  Pricing Context              │   │
│  │                           │  │                               │   │
│  │  Lenguaje: "listing",     │  │  Lenguaje: "price rule",      │   │
│  │  "variant", "attribute"   │  │  "tier", "dynamic rate"       │   │
│  │                           │  │                               │   │
│  │  Entidades: Product,      │  │  Entidades: PriceRule,        │   │
│  │  Category, Variant        │  │  PriceTier, PriceHistory      │   │
│  └───────────────────────────┘  └───────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  SUPPORTING DOMAIN                                                   │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  Orders Context  │  │  Inventory       │  │  Shipping        │  │
│  │                  │  │  Context         │  │  Context         │  │
│  │  "order",        │  │                  │  │                  │  │
│  │  "line item",    │  │  "stock unit",   │  │  "shipment",     │  │
│  │  "fulfillment"   │  │  "reservation",  │  │  "carrier",      │  │
│  │                  │  │  "restock"       │  │  "manifest"      │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  GENERIC DOMAIN (usar SaaS cuando sea posible)                       │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐    │
│  │  Identity &      │  │  Payment         │  │  Notifications   │    │
│  │  Access          │  │  Processing      │  │                  │    │
│  │  → Auth0 /       │  │  → Stripe /      │  │  → SendGrid /    │    │
│  │    Keycloak      │  │    PayPal        │  │    Twilio        │    │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘    │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐    │
│  │  Reviews         │  │  Search          │  │  Web UI          │    │
│  │                  │  │                  │  │                  │    │
│  │  → Yotpo /       │  │  → Algolia /     │  │  → Vercel /      │    │
│  │    Trustpilot    │  │    Elasticsearch │  │    Netlify       │    │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘    │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐                          │
│  │  Image           │  │  Image           │                          │
│  │  Compression     │  │  Security        │                          │
│  │  → Cloudinary /  │  │  → Cloudflare    │                          │
│  │    imgix         │  │    Images /      │                          │
│  │                  │  │    AWS Rekognition│                         │
│  └──────────────────┘  └──────────────────┘                          │
└──────────────────────────────────────────────────────────────────────┘
```

**ES:** El mapa de contextos (Context Map) muestra cómo se relacionan los Bounded Contexts entre sí:

**EN:** The Context Map shows how Bounded Contexts relate to each other:

```
Context Map de ShopFast:

  Catalog ──── publica evento ProductUpdated ────▶ Search
  Catalog ──── publica evento ProductUpdated ────▶ Pricing
  
  Orders ──── hace llamada sync ────▶ Catalog   (obtener detalles del producto)
  Orders ──── hace llamada sync ────▶ Pricing   (obtener precio actual)
  Orders ──── publica evento OrderCreated ────▶ Inventory
  Orders ──── publica evento OrderCreated ────▶ Payments
  Orders ──── publica evento OrderPaid ────▶ Shipping
  Orders ──── publica evento OrderPaid ────▶ Notifications

  Inventory ──── publica evento StockLow ────▶ Notifications
  Payments ──── publica evento PaymentFailed ────▶ Notifications
  Shipping ──── publica evento ShipmentUpdated ────▶ Notifications
```

---

### ✅ Pros / Advantages

| Pro | Descripción ES | Description EN |
|---|---|---|
| **Límites semánticamente ricos** | Los límites no solo separan código, sino que encapsulan un modelo de dominio con lenguaje y reglas propias. Son más precisos que las capacidades de negocio | Boundaries don't just separate code; they encapsulate a domain model with its own language and rules. More precise than business capabilities |
| **Resuelve ambigüedad de términos** | El mismo concepto (ej. "Product") puede significar cosas distintas en distintos contextos sin causar conflictos | The same concept (e.g., "Product") can mean different things in different contexts without causing conflicts |
| **Guía de inversión tecnológica** | Distinguir Core, Supporting y Generic indica dónde invertir (construir) y dónde ahorrar (comprar SaaS) | Distinguishing Core, Supporting, and Generic indicates where to invest (build) and where to save (buy SaaS) |
| **Equipos con ownership real** | Cada equipo posee un Bounded Context completo: su modelo, su DB, su API, sus reglas de negocio. Máxima autonomía | Each team owns a complete Bounded Context: its model, DB, API, business rules. Maximum autonomy |
| **Escalabilidad del equipo** | A medida que crece el negocio, se puede subdividir un Bounded Context en varios sin romper los demás | As the business grows, a Bounded Context can be subdivided into several without breaking the others |

---

### ❌ Contras / Disadvantages

| Contra | Descripción ES | Description EN |
|---|---|---|
| **Alta curva de aprendizaje** | DDD requiere que el equipo aprenda conceptos como Aggregate, Entity, Value Object, Domain Event, Saga. No es trivial | DDD requires the team to learn concepts like Aggregate, Entity, Value Object, Domain Event, Saga. Not trivial |
| **Requiere Event Storming o talleres** | Identificar Bounded Contexts correctamente requiere talleres colaborativos con expertos de negocio (Event Storming). Lleva días o semanas | Correctly identifying Bounded Contexts requires collaborative workshops with business experts (Event Storming). Takes days or weeks |
| **Puede ser sobre-ingeniería para equipos pequeños** | Un equipo de 5 personas probablemente no necesita DDD completo. La complejidad del proceso supera el beneficio | A team of 5 people probably doesn't need full DDD. The process complexity outweighs the benefit |
| **Límites pueden cambiar al profundizar** | Al conocer mejor el dominio, los Bounded Contexts iniciales pueden necesitar ser redefinidos, lo que implica refactorizar servicios ya en producción | As domain knowledge deepens, initial Bounded Contexts may need redefining, implying refactoring already-live services |

---

### Conclusión / Conclusion

> **ES:** La descomposición por dominio (DDD) es la estrategia más rigurosa y precisa para sistemas complejos con múltiples equipos y dominios ricos en reglas de negocio. Proporciona no solo los límites del servicio, sino también el modelo interno del servicio (aggregates, entities, domain events). Es la elección correcta cuando el dominio es complejo y el equipo tiene o puede adquirir conocimiento de DDD.
>
> **EN:** Decomposition by domain (DDD) is the most rigorous and precise strategy for complex systems with multiple teams and business-rule-rich domains. It provides not only service boundaries but also the internal model of the service (aggregates, entities, domain events). It's the right choice when the domain is complex and the team has or can acquire DDD knowledge.

---

## 3. Capacidades de negocio vs. Subdominios DDD / Business Capabilities vs. DDD Subdomains

### ¿En qué se parecen? / What Do They Have in Common?

**Español:**
Ambas estrategias parten del mismo principio fundamental: los microservicios deben organizarse alrededor del **negocio**, no de la tecnología. En la práctica, para muchos sistemas y muchos equipos, **ambas estrategias producen resultados muy similares** — los mismos servicios con los mismos límites aproximados. La diferencia está en la profundidad del análisis y el nivel de rigor del modelo interno.

**English:**
Both strategies share the same fundamental principle: microservices must be organized around the **business**, not technology. In practice, for many systems and many teams, **both strategies produce very similar results** — the same services with approximately the same boundaries. The difference lies in the depth of analysis and the rigor of the internal model.

```
Resultado típico en ShopFast:

  Por capacidades de negocio:    Por Bounded Context (DDD):
  
  User Service          ≈        Identity & Access Context
  Catalog Service       ≈        Catalog Context
  Order Service         ≈        Orders Context
  Payment Service       ≈        Payments Context
  Inventory Service     ≈        Inventory Context
  Shipping Service      ≈        Shipping Context
  Notification Service  ≈        Notifications Context
  Search Service        ≈        Search Context (infraestructura)
  Promotions Service    ≈        Promotions Context

  En la mayoría de casos: prácticamente el mismo resultado.
  In most cases: practically the same result.
```

---

### ¿En qué difieren? / Where Do They Differ?

| Dimensión | Capacidades de negocio | Subdominios / Bounded Contexts (DDD) |
|---|---|---|
| **Pregunta de partida** | ¿Qué hace el negocio? | ¿Cómo modela el negocio este concepto? |
| **Profundidad del modelo** | Define el límite del servicio (qué entra y qué no) | Define el límite + el modelo interno (aggregates, entities, value objects) |
| **Lenguaje** | Usa terminología funcional de negocio | Usa Ubiquitous Language — el vocabulario exacto de ese contexto |
| **Proceso de descubrimiento** | Entrevistas con stakeholders, análisis de procesos | Event Storming, Domain Modeling workshops (más estructurado) |
| **Guía de inversión** | No distingue prioridad entre servicios | Distingue Core / Supporting / Generic → guía dónde invertir |
| **Complejidad de aplicación** | Media — accesible para la mayoría de equipos | Alta — requiere conocimiento de DDD |
| **Mejor para** | Equipos con buen conocimiento de negocio, primer proyecto de microservicios | Sistemas complejos, múltiples equipos, dominio rico en reglas |

---

### Cuándo usar cada una / When to Use Each

```
¿Tu equipo conoce DDD?
        │
        ├── No ──▶  ¿El dominio es simple o moderado?
        │                   │
        │                   ├── Sí ──▶  Capacidades de negocio ✅
        │                   │           (claro, accesible, suficiente)
        │                   │
        │                   └── No ──▶  Aprender DDD básico primero
        │                               luego Bounded Contexts
        │
        └── Sí ──▶  ¿Cuántos equipos trabajan en el sistema?
                            │
                            ├── 1-3 ──▶  Capacidades de negocio ✅
                            │            (DDD puede ser sobre-ingeniería)
                            │
                            └── 4+  ──▶  Bounded Contexts (DDD) ✅
                                         (máxima autonomía entre equipos)
```

**Casos concretos / Concrete cases:**

| Situación | Recomendación |
|---|---|
| Startup con equipo de 8 personas migrando su primer monolito | Capacidades de negocio — rápido, claro, suficiente |
| Empresa con 5 squads autónomos y dominio financiero complejo | Bounded Contexts (DDD) — rigor y autonomía máxima |
| E-commerce mediano con 20 devs y reglas de negocio moderadas | Capacidades de negocio con vocabulario de DDD (híbrido práctico) |
| Empresa con múltiples líneas de producto y equipos en distintos países | Bounded Contexts (DDD) — necesario para manejar la escala organizacional |

---

### El enfoque híbrido / The Hybrid Approach

**Español:**
En la práctica, la mayoría de equipos exitosos usan un **enfoque híbrido**: identifican capacidades de negocio para definir los límites macro (qué servicios crear), y adoptan el vocabulario de DDD (Bounded Context, Ubiquitous Language, Domain Events) para diseñar el interior de cada servicio. Obtienen los beneficios de ambos sin la complejidad total de DDD desde el inicio.

**English:**
In practice, most successful teams use a **hybrid approach**: they identify business capabilities to define macro boundaries (which services to create), and adopt DDD vocabulary (Bounded Context, Ubiquitous Language, Domain Events) to design the interior of each service. They get the benefits of both without the full complexity of DDD from the start.

```
Enfoque híbrido / Hybrid approach:

  Fase 1 — Límites macro (Business Capabilities):
    "Necesitamos un Order Service, un Catalog Service,
     un Payment Service..." → rápido, claro, orientado a negocio

  Fase 2 — Modelo interno (DDD vocabulary):
    "Dentro del Order Service, el Aggregate raíz es Order.
     Un Order contiene OrderItems (Entities).
     La dirección de entrega es un Value Object.
     Cuando un Order se confirma, emite un OrderConfirmed Domain Event."

  Resultado: servicios bien delimitados con modelos internos sólidos,
  sin necesidad de un Event Storming de 3 días desde el día 1.
```

---

### Comparativa final / Final Comparison

| | Capacidades de negocio | Subdominios DDD | Híbrido |
|---|---|---|---|
| **Velocidad de aplicación** | ⚡ Alta | 🐢 Baja | ⚡ Media-Alta |
| **Precisión del modelo** | ⚠️ Media | ✅ Alta | ✅ Media-Alta |
| **Guía de inversión** | ❌ No | ✅ Core/Supporting/Generic | ⚠️ Parcial |
| **Curva de aprendizaje** | ✅ Baja | ❌ Alta | ✅ Media |
| **Resultado en sistema simple** | ✅ Suficiente | ⚠️ Sobre-ingeniería | ✅ Ideal |
| **Resultado en sistema complejo** | ⚠️ Puede quedarse corto | ✅ Ideal | ✅ Muy bueno |
| **Recomendado para** | Equipos nuevos en microservicios | Sistemas enterprise maduros | La mayoría de casos reales |

> **ES:** No existe una respuesta universalmente correcta. La mejor estrategia de descomposición es la que el equipo puede aplicar consistentemente, alineada con la complejidad real del dominio y la madurez organizacional del negocio.
>
> **EN:** There is no universally correct answer. The best decomposition strategy is the one the team can consistently apply, aligned with the real complexity of the domain and the organizational maturity of the business.