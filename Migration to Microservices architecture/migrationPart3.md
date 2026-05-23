# Migración a Microservicios — Pasos, Consejos y Patrones / Migration to Microservices — Steps, Tips and Patterns

---

## 1. Por dónde empezar la migración / Where to Start the Migration

### El enfoque Big Bang / The Big Bang Approach

**Español:**
El **enfoque Big Bang** consiste en detener todo el desarrollo de nuevas funcionalidades, reescribir el sistema completo como microservicios desde cero y hacer el switch en un único evento de puesta en producción. Es el enfoque más intuitivo y también el más peligroso.

**English:**
The **Big Bang approach** consists of stopping all new feature development, rewriting the entire system as microservices from scratch, and switching everything over in a single go-live event. It's the most intuitive approach and also the most dangerous.

```
Enfoque Big Bang / Big Bang Approach:

  Hoy / Today           Meses de reescritura    Día de go-live
       │                  (sin entregas)              │
       ▼                                              ▼
  ┌──────────┐  ════════════════════════════▶  ┌────────────────┐
  │ Monolito │  todo el equipo en pausa de     │ Microservicios │
  │ en prod  │  features → reescribiendo       │ nuevos en prod │
  └──────────┘                                 └────────────────┘

  Riesgos / Risks:
  ✗ Meses sin entregar valor al negocio
  ✗ El nuevo sistema nunca tiene el mismo comportamiento que el monolito
  ✗ Si algo sale mal el día del go-live, el rollback es catastrófico
  ✗ El conocimiento del dominio se pierde durante la reescritura
  ✗ Tendencia a sobre-diseñar el nuevo sistema desde cero (over-engineering)
```

> **ES:** El Big Bang rara vez funciona. La mayoría de proyectos de reescritura total fracasan, se cancelan a mitad o producen un sistema peor que el original. **No uses este enfoque.**
>
> **EN:** Big Bang rarely works. Most total rewrite projects fail, get cancelled halfway, or produce a worse system than the original. **Do not use this approach.**

---

### El plan correcto / The Correct Plan

#### Enfoque incremental y continuo / Incremental and Continuous Approach

**Español:**
La alternativa al Big Bang es una migración **incremental y continua**: el monolito sigue en producción sirviendo tráfico real mientras se extraen servicios uno a uno. No se para el desarrollo, no hay un "día D". El sistema evoluciona de forma continua.

**English:**
The alternative to Big Bang is an **incremental and continuous** migration: the monolith stays in production serving real traffic while services are extracted one by one. Development does not stop, there is no "D-Day". The system evolves continuously.

```
Beneficios del enfoque incremental / Benefits of the incremental approach:

  ✓ No hard deadlines necessary
    → No hay una fecha fija de "go-live total" que genere presión artificial
    → La migración avanza servicio a servicio, sin fecha límite crítica

  ✓ Consistent, visible and measurable progress
    → Cada servicio extraído es un entregable concreto y demostrable
    → El progreso es visible para el negocio: X de N servicios migrados

  ✓ Business is not disrupted
    → El monolito sigue sirviendo tráfico real durante toda la migración
    → Los usuarios no perciben ningún corte ni degradación del servicio

  ✓ Exceeding the time estimate is not a problem
    → Si la migración tarda más de lo previsto, el negocio sigue operando
    → No hay consecuencias catastróficas por retrasos — el sistema funciona
```

#### Identificar los mejores candidatos / Identify the Best Candidates

**Español:**
No todos los módulos son igual de fáciles o valiosos de extraer primero. Pogrebinsky identifica tres criterios para priorizar qué extraer primero:

**English:**
Not all modules are equally easy or valuable to extract first. Pogrebinsky identifies three criteria to prioritize what to extract first:

```
Criterios de priorización / Prioritization criteria:

  a) Áreas con más cambios / desarrollo de features
     → Son las más dolorosas en el monolito (deploys lentos, conflicts)
     → Extraerlas primero entrega valor de negocio rápidamente
     → Ejemplo ShopFast: Catalog Service (cambios frecuentes de producto)

  b) Componentes con alta demanda de escalabilidad
     → El monolito no puede escalar partes de forma independiente
     → Extraer estos componentes desbloquea el escalado granular
     → Ejemplo ShopFast: Search Service (picos de tráfico en búsquedas)

  c) Componentes con menos deuda técnica
     → Más fáciles de extraer limpiamente (menos código enredado)
     → Menor riesgo de introducir bugs durante la extracción
     → Ejemplo ShopFast: Notification Service (bien aislado, pocos acoplamientos)
```

> **ES:** La estrategia óptima combina los tres criterios: empezar por servicios que sean *fáciles de extraer* **y** *dolorosos de operar en el monolito*. Esto genera victorias tempranas que mantienen el momentum del equipo.
>
> **EN:** The optimal strategy combines all three criteria: start with services that are *easy to extract* **and** *painful to operate in the monolith*. This generates early wins that maintain team momentum.

**Español:**
La alternativa al Big Bang es una migración **incremental y paralela**: el monolito sigue en producción sirviendo tráfico real mientras se extraen servicios uno a uno. El plan tiene dos pasos previos a cualquier línea de código nueva:

**English:**
The alternative to Big Bang is an **incremental and parallel** migration: the monolith stays in production serving real traffic while services are extracted one by one. The plan has two steps before writing any new code:

#### Paso 1: Mapear los límites de los microservicios / Map Out the Microservice Boundaries

**ES:** Antes de extraer un solo servicio, documentar el mapa completo de hacia dónde se quiere llegar. Usar las estrategias de descomposición (capacidades de negocio o DDD) para identificar todos los servicios candidatos, sus responsabilidades y sus dependencias. Este mapa es el "plano de arquitectura" de la migración.

**EN:** Before extracting a single service, document the complete map of where you want to end up. Use decomposition strategies (business capabilities or DDD) to identify all candidate services, their responsibilities, and their dependencies. This map is the migration's "architecture blueprint."

```
Mapa de límites para ShopFast / Boundary map for ShopFast:

  ┌──────────────────────────────────────────────────────────────┐
  │  Servicio          │ Responsabilidad      │ Dependencias     │
  ├────────────────────┼──────────────────────┼──────────────────┤
  │  User Service      │ auth, perfiles, roles│ ninguna          │
  │  Catalog Service   │ productos, categorías│ ninguna          │
  │  Order Service     │ pedidos, estado      │ Catalog, Pricing │
  │  Payment Service   │ cobros, reembolsos   │ Order            │
  │  Inventory Service │ stock, reservas      │ Catalog          │
  │  Shipping Service  │ envíos, tracking     │ Order, Inventory │
  │  Notification Svc  │ email, SMS, push     │ ninguna          │
  │  Search Service    │ búsqueda full-text   │ Catalog          │
  └──────────────────────────────────────────────────────────────┘

  → Servicios sin dependencias = primeros candidatos a extraer
  → Servicios con muchas dependencias = extraer al final
```

#### Paso 2: Detener nuevas funcionalidades en el monolito / Stop New Features in the Monolith

**ES:** Una vez iniciada la migración, acordar con el negocio que **no se desarrollan nuevas funcionalidades en el monolito**. Toda nueva funcionalidad se desarrolla directamente en el microservicio correspondiente. Esto evita que el monolito siga creciendo mientras se intenta reducir.

**EN:** Once migration has begun, agree with the business that **no new features are developed in the monolith**. All new functionality is developed directly in the corresponding microservice. This prevents the monolith from continuing to grow while you're trying to shrink it.

```
❌ Sin esta regla / Without this rule:
  
  Mes 1: extraes Notification Service del monolito ✅
  Mes 1: el equipo de Marketing añade "campañas push" al monolito ❌
  Mes 2: el monolito creció, Notification Service quedó desactualizado
  Resultado: migración interminable — el monolito nunca encoge

✅ Con esta regla / With this rule:
  
  Mes 1: extraes Notification Service del monolito ✅
  Mes 1: "campañas push" se construye directamente en Notification Service ✅
  Resultado: el monolito encoge, el nuevo servicio crece con valor real
```

---

## 2. Preparando la migración / Preparing for the Migration

**Español:**
Antes de extraer el primer servicio, el equipo debe preparar la infraestructura y los procesos que harán que la migración sea sostenible. Intentar migrar sin esta preparación produce caos operacional.

**English:**
Before extracting the first service, the team must prepare the infrastructure and processes that will make the migration sustainable. Attempting to migrate without this preparation produces operational chaos.

### Infraestructura necesaria / Required Infrastructure

```
Lista de preparación / Preparation checklist:

  □ API Gateway
    → Punto de entrada único para redirigir tráfico al monolito o a nuevos servicios
    → Ejemplos: AWS API Gateway, Kong, Nginx, Traefik

  □ CI/CD pipeline por servicio
    → Cada nuevo microservicio necesita su propio pipeline desde el día 1
    → Sin esto, los deploys serán manuales y lentos — peor que el monolito

  □ Observabilidad centralizada
    → Logs centralizados (ELK Stack, Datadog, CloudWatch)
    → Métricas y dashboards (Prometheus + Grafana)
    → Distributed tracing (Jaeger, Zipkin) para rastrear requests entre servicios
    → Sin esto, depurar problemas en producción será imposible

  □ Service discovery
    → Mecanismo para que los servicios se encuentren entre sí dinámicamente
    → Ejemplos: Kubernetes DNS, Consul, AWS Cloud Map

  □ Containerización
    → Cada servicio corre en su propio contenedor (Docker)
    → Orquestador: Kubernetes (producción) o Docker Compose (desarrollo local)

  □ Feature flags
    → Capacidad de activar/desactivar el nuevo servicio por porcentaje de tráfico
    → Permite rollback instantáneo si algo falla sin hacer un nuevo deploy
```

### Pruebas de contrato / Contract Testing

**Español:**
Con múltiples servicios comunicándose, las pruebas de integración tradicionales no escalan. En su lugar, se usan **pruebas de contrato** (contract tests): cada servicio define el contrato de su API (qué datos envía y recibe) y los tests verifican que ambos lados del contrato son compatibles antes del deploy.

**English:**
With multiple services communicating, traditional integration tests don't scale. Instead, **contract tests** are used: each service defines its API contract (what data it sends and receives), and tests verify that both sides of the contract are compatible before deployment.

```
Consumer-Driven Contract Testing (Pact):

  Order Service (consumer)          Catalog Service (provider)
       │                                    │
       │  define contrato:                  │
       │  "cuando llamo GET /products/:id   │
       │   espero recibir                   │
       │   { id, name, price }"             │
       │                                    │
       └──── contrato publicado ───────────▶│
                                            │  verifica que su API
                                            │  cumple el contrato ✅
                                            │
  Si Catalog Service cambia su API sin respetar el contrato → test falla
  antes del deploy → nunca llega a producción roto
```

---

## 3. Ejecutar la migración con el patrón Strangler Fig / Execute the Migration Using the Strangler Fig Pattern

### Definición / Definition

**Español:**
El **patrón Strangler Fig** (higuera estranguladora) toma su nombre de una planta tropical que crece alrededor de un árbol huésped hasta reemplazarlo completamente sin que el árbol huésped muera de golpe. En la migración de software, el nuevo sistema (microservicios) crece alrededor del monolito, interceptando gradualmente su tráfico, hasta que el monolito queda vacío y puede eliminarse.

**English:**
The **Strangler Fig pattern** takes its name from a tropical plant that grows around a host tree until it completely replaces it — without the host tree dying all at once. In software migration, the new system (microservices) grows around the monolith, gradually intercepting its traffic, until the monolith is empty and can be removed.

```
Evolución del Strangler Fig en ShopFast:

  Semana 1-4: solo el monolito
  ┌─────────────────────────────────────┐
  │            Monolito                 │ ← todo el tráfico
  │  Users+Catalog+Orders+Payments+...  │
  └─────────────────────────────────────┘

  Semana 5-8: primer servicio extraído
  ┌───────────────────────────────────────────────────┐
  │                   API Gateway                     │
  └──────┬─────────────────────────────────┬──────────┘
         │ /api/notifications/*             │ todo lo demás
         ▼                                  ▼
  ┌──────────────┐              ┌───────────────────────────┐
  │  Notification│              │         Monolito          │
  │   Service ✅  │              │  Users+Catalog+Orders+... │
  └──────────────┘              └───────────────────────────┘

  Mes 3-6: varios servicios extraídos
  ┌─────────────────────────────────────────────────────────────┐
  │                        API Gateway                          │
  └──┬───────────┬──────────────┬──────────────────┬───────────┘
     │           │              │                   │
     ▼           ▼              ▼                   ▼
  ┌──────┐  ┌────────┐  ┌──────────────┐  ┌───────────────────┐
  │Notif.│  │ Search │  │   Catalog    │  │    Monolito       │
  │  ✅  │  │   ✅   │  │     ✅       │  │  Users+Orders+... │
  └──────┘  └────────┘  └──────────────┘  └───────────────────┘
                                               (cada vez más pequeño)

  Mes 12+: migración completa
  ┌────────────────────────────────────────────────────────────┐
  │                        API Gateway                         │
  └──┬──────────┬──────────┬──────────┬──────────┬────────────┘
     ▼          ▼          ▼          ▼          ▼
  ┌──────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
  │Users │ │Catalog │ │Orders  │ │Payment │ │  ...   │
  │  ✅  │ │   ✅   │ │   ✅   │ │   ✅   │ │   ✅   │
  └──────┘ └────────┘ └────────┘ └────────┘ └────────┘
                    Monolito eliminado ✅
```

---

### Los 4 pasos del Strangler Fig / The 4 Steps of Strangler Fig

#### Paso 1: Identificar y aislar el módulo / Identify and Isolate the Module

**ES:** Dentro del monolito, crear una interfaz (interface/facade) clara alrededor del módulo a extraer. Todo el código del monolito que necesite esa funcionalidad debe llamar a la interfaz, no directamente a la implementación. Esto prepara el módulo para ser reemplazado sin romper el resto del monolito.

**EN:** Inside the monolith, create a clear interface/facade around the module to be extracted. All monolith code that needs that functionality must call the interface, not directly the implementation. This prepares the module to be replaced without breaking the rest of the monolith.

```
Antes / Before:
  orderService.js:
    const template = notificationTemplates.getById(templateId)  // acceso directo
    await emailClient.send(user.email, template)                 // acceso directo

Después de aislar / After isolating:
  orderService.js:
    await notificationFacade.sendOrderConfirmation(userId, orderId)
    // el facade oculta la implementación — puede ser local o remota
```

#### Paso 2: Construir el nuevo microservicio / Build the New Microservice

**ES:** Construir el nuevo servicio de forma completamente independiente: su propio repositorio, su propia base de datos, su propio pipeline de CI/CD. El servicio debe estar completamente funcional y probado antes de recibir tráfico real.

**EN:** Build the new service completely independently: its own repository, its own database, its own CI/CD pipeline. The service must be fully functional and tested before receiving real traffic.

```
Checklist del nuevo servicio / New service checklist:
  □ Repositorio Git propio
  □ Dockerfile + docker-compose para desarrollo local
  □ Tests unitarios e integración (>80% coverage)
  □ Pipeline CI/CD (build → test → deploy a staging)
  □ Documentación de API (OpenAPI/Swagger)
  □ Health check endpoint (/health)
  □ Logs estructurados (JSON) conectados a la plataforma central
  □ Métricas expuestas (/metrics para Prometheus)
  □ Base de datos propia (migrada o nueva, según el caso)
```

#### Paso 3: Redirigir el tráfico via API Gateway / Redirect Traffic via API Gateway

**ES:** Con el nuevo servicio desplegado y probado en staging, configurar el API Gateway para que las rutas correspondientes apunten al nuevo servicio en lugar del monolito. El monolito sigue corriendo y maneja todo lo demás sin cambios.

**EN:** With the new service deployed and tested in staging, configure the API Gateway so the corresponding routes point to the new service instead of the monolith. The monolith keeps running and handles everything else unchanged.

```
Configuración del API Gateway (ejemplo Nginx):

  # Antes
  location /api/ {
    proxy_pass http://monolith:3000;
  }

  # Después de extraer Notification Service
  location /api/notifications/ {
    proxy_pass http://notification-service:4001;   ← nuevo servicio
  }
  location /api/ {
    proxy_pass http://monolith:3000;               ← todo lo demás
  }
```

#### Paso 4: Retirar el módulo del monolito / Remove the Module from the Monolith

**ES:** Una vez que el nuevo servicio lleva semanas estable en producción con tráfico real, eliminar el código correspondiente del monolito. El monolito se hace más pequeño. Repetir desde el Paso 1 con el siguiente módulo.

**EN:** Once the new service has been running stably in production with real traffic for weeks, delete the corresponding code from the monolith. The monolith gets smaller. Repeat from Step 1 with the next module.

```
Orden recomendado de extracción para ShopFast:

  Ronda 1 (servicios periféricos, sin dependencias):
    1. Notification Service  → solo recibe eventos, sin dependencias salientes
    2. Search Service        → solo consume datos de Catalog (read-only)
    3. Reports Service       → solo lee datos, no los modifica

  Ronda 2 (servicios de soporte con dependencias simples):
    4. Catalog Service       → base de datos propia, pocos consumidores
    5. Inventory Service     → depende de Catalog (ya extraído)
    6. User Service          → autenticación, bien aislado

  Ronda 3 (servicios core, máxima complejidad):
    7. Shipping Service      → depende de Orders e Inventory
    8. Payment Service       → alta criticidad, extraer con mucho cuidado
    9. Order Service         → el más complejo, extraer al final
```

---

## 4. Consejos para una migración fluida / Tips to Ensure a Smooth Migration

### Consejo 1: Nunca compartir la base de datos / Never Share the Database

**Español:**
El error más común durante la migración es permitir que el nuevo microservicio acceda directamente a la base de datos del monolito. Es tentador porque parece más rápido, pero crea un acoplamiento invisible que hace la separación definitiva mucho más difícil después.

**English:**
The most common mistake during migration is allowing the new microservice to access the monolith's database directly. It's tempting because it seems faster, but it creates invisible coupling that makes the final separation much harder later.

```
❌ Antipatrón: DB compartida / Anti-pattern: Shared DB
  
  Notification Service ──────────────▶ Monolith DB
  (accede a tabla users para obtener email)

  Problema: si el monolito cambia el esquema de users,
  Notification Service se rompe. No hay autonomía real.

✅ Correcto: API call o evento / Correct: API call or event
  
  Order Service ──publica evento OrderConfirmed──▶ Message Bus
                                                         │
                                    Notification Service ◀┘
                                    (recibe el email en el payload del evento)
  
  Notification Service no necesita acceder a ninguna DB del monolito.
```

### Consejo 2: Doble escritura durante la transición / Dual Write During Transition

**Español:**
Cuando se separa la base de datos de un módulo, hay un período donde tanto el monolito como el nuevo servicio necesitan los mismos datos. La técnica de **doble escritura** (dual write) consiste en escribir en ambas bases de datos simultáneamente durante la transición, hasta que el monolito deje de necesitar esos datos.

**English:**
When splitting a module's database, there is a period where both the monolith and the new service need the same data. The **dual write** technique consists of writing to both databases simultaneously during the transition, until the monolith no longer needs that data.

```
Fases de separación de DB / DB separation phases:

  Fase A — Solo monolito (antes de la migración):
    Write ──▶ Monolith DB
    Read  ◀── Monolith DB

  Fase B — Doble escritura (durante la transición):
    Write ──▶ Monolith DB  (fuente de verdad)
    Write ──▶ New Service DB (copia sincronizada)
    Read  ◀── Monolith DB  (el monolito sigue leyendo su DB)
    Read  ◀── New Service DB (el nuevo servicio lee su propia DB)

  Fase C — Monolito lee del nuevo servicio (validación):
    Write ──▶ New Service DB (nueva fuente de verdad)
    Write ──▶ Monolith DB  (solo para fallback)
    Read  ◀── New Service DB

  Fase D — Monolito ya no necesita esos datos:
    Write ──▶ New Service DB ✅
    Read  ◀── New Service DB ✅
    Monolith DB tables → DROP TABLE (eliminadas)
```

### Consejo 3: Usar feature flags para el rollback / Use Feature Flags for Rollback

**Español:**
Cada vez que se redirige tráfico a un nuevo servicio, implementar un **feature flag** que permita volver al monolito en segundos sin necesidad de un nuevo deploy. Si el nuevo servicio tiene un problema en producción, el rollback es cambiar un valor en la configuración.

**English:**
Every time traffic is redirected to a new service, implement a **feature flag** that allows returning to the monolith in seconds without needing a new deployment. If the new service has a problem in production, rollback is just changing a configuration value.

```
Implementación con feature flags:

  API Gateway config:
    NOTIFICATION_SERVICE_ENABLED = true   ← cambiar a false para rollback

  Lógica del gateway:
    if (NOTIFICATION_SERVICE_ENABLED && route.startsWith('/api/notifications')) {
      proxy → Notification Service
    } else {
      proxy → Monolith
    }

  Rollback en caso de incidente:
    NOTIFICATION_SERVICE_ENABLED = false  ← cambia en segundos, sin deploy
    Todo el tráfico vuelve al monolito inmediatamente.
```

### Consejo 4: Medir antes, durante y después / Measure Before, During and After

**Español:**
Definir métricas de éxito **antes** de iniciar la migración para poder comparar objetivamente si la migración está mejorando las cosas. Sin métricas base, es imposible saber si los cambios son beneficiosos o perjudiciales.

**English:**
Define success metrics **before** starting the migration to objectively compare whether the migration is improving things. Without baseline metrics, it's impossible to know if changes are beneficial or harmful.

```
Métricas a medir / Metrics to measure:

  Antes de migrar (baseline):        Objetivo post-migración:
  ┌──────────────────────────────────────────────────────────┐
  │  Métrica              │  Baseline   │  Objetivo          │
  ├───────────────────────┼─────────────┼────────────────────┤
  │  Tiempo de deploy     │  45 min     │  < 5 min           │
  │  Frecuencia de deploy │  1x/semana  │  múltiples/día     │
  │  Latencia p99 /orders │  850ms      │  < 400ms           │
  │  Error rate           │  0.8%       │  < 0.2%            │
  │  MTTR (recovery time) │  2 horas    │  < 15 min          │
  │  Onboarding time      │  3 meses    │  < 3 semanas       │
  └──────────────────────────────────────────────────────────┘

  Si después de extraer un servicio las métricas empeoran →
  revisar el diseño del servicio antes de continuar con el siguiente.
```

### Consejo 5: Migrar en equipo, no en solitario / Migrate as a Team, Not Solo

**Español:**
Cada microservicio debe tener un **equipo dueño** (owning team) claramente definido desde antes de su extracción. El equipo que extrajo el módulo del monolito debe ser el mismo que lo opera en producción. Esto crea responsabilidad real y acelera el aprendizaje operacional.

**English:**
Each microservice must have a clearly defined **owning team** before its extraction. The team that extracted the module from the monolith should be the same team that operates it in production. This creates real accountability and accelerates operational learning.

```
Estructura de equipos en ShopFast durante la migración:

  Squad Catalog    → owner: Catalog Service
  Squad Orders     → owner: Order Service + Inventory Service
  Squad Payments   → owner: Payment Service
  Squad Platform   → owner: API Gateway + CI/CD + Observabilidad
  Squad Growth     → owner: Search Service + Promotions Service
  Squad Users      → owner: User Service + Notification Service

  Regla: el squad que construye el servicio recibe las alertas de producción.
  Rule: the squad that builds the service receives the production alerts.
```

---

### Resumen: Lista de verificación de migración / Migration Checklist Summary

```
ANTES DE EMPEZAR / BEFORE STARTING:
  □ Mapa de Bounded Contexts / límites definido
  □ Métricas baseline medidas
  □ API Gateway instalado y configurado
  □ Pipeline de CI/CD listo para nuevos servicios
  □ Plataforma de observabilidad operativa (logs, métricas, traces)
  □ Acuerdo con negocio: no nuevas features en el monolito

POR CADA SERVICIO EXTRAÍDO / FOR EACH EXTRACTED SERVICE:
  □ Módulo aislado con facade en el monolito
  □ Nuevo servicio con DB propia
  □ Tests de contrato escritos y pasando
  □ Deploy en staging validado
  □ Feature flag configurado para rollback
  □ Dashboard de monitoreo creado
  □ Runbook de incidentes escrito
  □ Tráfico redirigido gradualmente (10% → 50% → 100%)
  □ Módulo eliminado del monolito después de N semanas estable

AL FINALIZAR / WHEN COMPLETE:
  □ Monolito vacío y eliminado
  □ Todas las métricas objetivo alcanzadas o superadas
  □ Cada servicio con su equipo dueño operándolo
  □ Documentación de arquitectura actualizada
```