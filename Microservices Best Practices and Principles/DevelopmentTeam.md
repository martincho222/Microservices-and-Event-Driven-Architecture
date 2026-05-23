# Autonomía Estructurada para Equipos de Desarrollo / Structured Autonomy for Development Teams

---

## 1. Problemas con la autonomía total del equipo / Problems with Full Team Autonomy

**Español:**
Una de las promesas de los microservicios es que cada equipo puede tomar sus propias decisiones tecnológicas. Sin embargo, si esta autonomía es **irrestricta**, la organización se fragmenta rápidamente y los costos superan los beneficios.

**English:**
One of the promises of microservices is that each team can make its own technology decisions. However, if this autonomy is **unrestricted**, the organization quickly fragments and the costs outweigh the benefits.

```
Escenario: autonomía total sin límites / Full autonomy without limits (ShopFast):

  Squad Orders    → Java + Spring Boot + PostgreSQL + Jenkins + Grafana
  Squad Catalog   → Node.js + Express + MongoDB + GitHub Actions + Datadog
  Squad Payments  → Python + FastAPI + MySQL + CircleCI + New Relic
  Squad Shipping  → Go + Gin + CockroachDB + GitLab CI + Prometheus
  Squad Search    → Kotlin + Ktor + Elasticsearch + Travis CI + Splunk
  Squad Users     → Ruby + Rails + SQLite + Bamboo + CloudWatch

  Problemas resultantes / Resulting problems:
  ✗ Un ingeniero de Squad Orders no puede ayudar a Squad Catalog en una emergencia
  ✗ El equipo de Platform debe soportar 5 lenguajes, 6 DBs, 5 CI/CD tools
  ✗ Onboarding de un nuevo ingeniero: ¿a qué squad va? ¿qué stack aprende?
  ✗ Auditoría de seguridad: cada servicio tiene estándares distintos
  ✗ Costo de licencias y tooling multiplicado por N
```

**Español:**
Los problemas concretos de la autonomía total son:

**English:**
The concrete problems of full autonomy are:

```
  ✗ Silos de conocimiento
     → El conocimiento queda atrapado en cada equipo
     → Imposible rotar ingenieros entre equipos
     → Bus factor extremadamente alto: si el experto se va, el conocimiento se pierde

  ✗ Fragmentación tecnológica
     → El equipo de plataforma/infraestructura debe soportar N stacks distintos
     → Cada nueva tecnología añade deuda operacional para toda la organización

  ✗ Inconsistencia en estándares de seguridad
     → Cada equipo implementa auth, logging, TLS a su manera
     → Superficie de ataque impredecible y difícil de auditar

  ✗ Integración caótica entre servicios
     → Sin estándares de API, cada servicio tiene su propio formato de respuesta
     → Los consumidores deben adaptar su código para cada proveedor

  ✗ Onboarding imposible de escalar
     → Un nuevo ingeniero debe aprender un stack diferente por cada servicio
       en el que trabaje — tiempo de productividad se multiplica

  ✗ Costo inicial de infraestructura (Upfront cost of infrastructure)
     → Cada equipo que elige una nueva tecnología debe levantar toda su
       infraestructura desde cero: clusters, pipelines, configuración de red
     → Ese costo lo paga el equipo de Platform, no el equipo que eligió la tech
     → Con 6 equipos y 6 stacks distintos → 6 veces el costo de setup inicial

  ✗ Costo de mantenimiento de infraestructura (Infrastructure maintenance cost)
     → Cada tecnología en producción requiere: patches de seguridad, upgrades
       de versión, monitoreo especializado, runbooks de incidentes
     → Un cluster de Kafka, uno de RabbitMQ y uno de Redis para mensajería
       = 3 veces el trabajo operacional vs estandarizar en uno solo
     → El costo crece linealmente con cada nueva tecnología adoptada

  ✗ Curva de aprendizaje y overhead (Learning curve and overhead)
     → Cada nueva tecnología requiere tiempo de aprendizaje antes de ser
       productiva en producción
     → Los ingenieros senior deben invertir tiempo en evaluar, experimentar
       y documentar cada nueva herramienta adoptada por el equipo
     → En una crisis de producción, el tiempo de diagnóstico es mayor
       si el stack no es familiar para el equipo de guardia

  ✗ API no uniforme entre servicios (Non-uniform API)
     → Sin estándares, cada servicio diseña su API de forma diferente:
       • Order Service:   GET /orders/:id        → { order_id, items: [...] }
       • Catalog Service: GET /product?id=123    → { productData: { ... } }
       • User Service:    POST /getUser          → { userData: [...] }
     → Los clientes (frontend, otros servicios) deben adaptarse a cada contrato
     → Imposible generar SDKs o documentación unificada
     → Cada integración nueva requiere leer la documentación desde cero
```

> **ES:** La autonomía total no es un objetivo — es un anti-patrón. El objetivo es **autonomía estructurada**: máxima libertad dentro de límites que protegen la coherencia organizacional.
>
> **EN:** Full autonomy is not a goal — it's an anti-pattern. The goal is **structured autonomy**: maximum freedom within boundaries that protect organizational coherence.

---

## 2. Los 3 niveles de autonomía del equipo / 3 Tiers of Development Team Autonomy

**Español:**
Pogrebinsky propone organizar las decisiones de un equipo en tres niveles según el grado de autonomía que se les concede. Cuanto más impacto tiene una decisión en otros equipos, menos autonomía individual debe haber.

**English:**
Pogrebinsky proposes organizing team decisions into three tiers based on the degree of autonomy granted. The more impact a decision has on other teams, the less individual autonomy there should be.

```
Pirámide de autonomía / Autonomy pyramid:

         ┌─────────────────────────────────────┐
         │       TIER 1: Autonomía total        │  ← decisiones internas
         │   Full autonomy (dentro del servicio)│     solo afectan al equipo
         └──────────────────────────────────────┘
        ┌────────────────────────────────────────┐
        │    TIER 2: Autonomía guiada            │  ← elegir dentro de
        │    Guided autonomy (opciones aprobadas)│     un menú definido
        └────────────────────────────────────────┘
       ┌──────────────────────────────────────────┐
       │    TIER 3: Sin autonomía                 │  ← estándares
       │    No autonomy (estándares obligatorios) │     no negociables
       └──────────────────────────────────────────┘
```

### Tier 1 — Autonomía total / Full Autonomy

**Español:**
Decisiones que solo afectan al interior del servicio. El equipo tiene control completo y no necesita coordinarse con nadie.

**English:**
Decisions that only affect the interior of the service. The team has full control and doesn't need to coordinate with anyone.

```
✅ Decisiones de Tier 1 (solo afectan al equipo):

  • Diseño interno del código (patrones, arquitectura interna, capas)
  • Estructura de carpetas y módulos del repositorio
  • Modelos de datos internos (esquema de la DB propia)
  • Algoritmos y lógica de negocio
  • Frecuencia de deploys (el equipo decide cuándo despliega)
  • Librerías internas que no se exponen a otros servicios
  • Cobertura de tests y estrategia de testing interna
  • Documentation

  Ejemplo ShopFast / ShopFast example:
  Squad Orders decide usar el patrón Repository internamente → nadie más
  necesita saberlo ni aprobarlo.
```

### Tier 2 — Autonomía guiada / Guided Autonomy / Freedom with Boundaries

**Español:**
Decisiones que afectan a otros equipos o a la plataforma, pero donde se permite elegir entre un conjunto de opciones pre-aprobadas por la organización. El equipo tiene libertad, pero dentro de un menú definido.

**English:**
Decisions that affect other teams or the platform, but where teams can choose from a set of pre-approved options. The team has freedom, but within a defined menu.

```
✅ Decisiones de Tier 2 (elegir de opciones aprobadas):

  Categoría          │ Opciones aprobadas (ejemplo ShopFast)
  ───────────────────┼────────────────────────────────────────────
  Lenguaje           │ Java, Go, Python, Node.js, Kotlin
  Base de datos      │ PostgreSQL, MongoDB, Redis, Elasticsearch
  Framework HTTP     │ Spring Boot, Express, FastAPI, Gin
  Message broker     │ Kafka, RabbitMQ (solo estos dos)
  Cache              │ Redis, Memcached

  → Cada equipo elige lo que mejor se adapta a su dominio
  → Pero solo de la lista aprobada — el equipo de Platform ya sabe
    cómo soportar, monitorear y operar esas tecnologías
```

### Tier 3 — Sin autonomía / No Autonomy

**Español:**
Decisiones que afectan a toda la organización o que tienen implicaciones de seguridad, compliance u operacionales críticas. No son negociables — todos los equipos deben seguirlas.

**English:**
Decisions that affect the entire organization or have critical security, compliance, or operational implications. These are non-negotiable — all teams must follow them.

```
❌ Decisiones de Tier 3 (obligatorias para todos):

  • Formato de logs: JSON estructurado con campos estándar (timestamp,
    service, level, traceId, spanId) → sin excepciones
  • Autenticación entre servicios: mTLS o JWT firmado con la clave corporativa
  • API Gateway: todos los servicios expuestos DEBEN ir detrás del gateway
  • Distributed tracing: OpenTelemetry con Jaeger — obligatorio en todos los servicios
  • CI/CD: el pipeline debe incluir SAST (análisis estático de seguridad)
  • Versionado de API: semver + backward compatibility obligatoria
  • Health checks: /health y /ready endpoints con formato estándar
  • Secrets management: solo via Vault o AWS Secrets Manager, nunca en código
  • API guidelines and best practices
  • Security and data compliance

  → Estos estándares permiten que el equipo de Platform opere N servicios
    como si fueran uno solo desde la perspectiva de observabilidad y seguridad
```

---

## 3. Factores que definen los límites de autonomía / Factors in Team Autonomy Boundaries

**Español:**
Los límites entre los tres niveles no son fijos para todas las organizaciones. Varios factores determinan cuánta autonomía es apropiada en cada contexto.

**English:**
The boundaries between the three tiers are not fixed for all organizations. Several factors determine how much autonomy is appropriate in each context.

### Factor 1: Tamaño y madurez del equipo / Team Size and Maturity

| Situación | Autonomía recomendada | Razonamiento |
|---|---|---|
| Equipo pequeño (2-4 personas) | Menor | Menos capidad para soportar múltiples stacks |
| Equipo senior experimentado | Mayor | Puede tomar decisiones tecnológicas con bajo riesgo |
| Equipo nuevo / recién formado | Menor | Necesita guardarrails mientras madura |
| Equipo con dominio complejo (Payments) | Menor en Tier 3 | Mayor criticidad → más estándares obligatorios |

**Pros de dar más autonomía a equipos maduros:**
- Aprovecha el expertise del equipo para elegir la mejor herramienta
- Mayor motivación y ownership al tener control real

**Contras de dar más autonomía a equipos maduros:**
- Requiere confianza explícita y mecanismos de revisión
- Puede generar resentimiento en otros equipos con menos autonomía

### Factor 2: Naturaleza del servicio / Nature of the Service

```
  Tipo de servicio     │ Nivel de autonomía │ Razón
  ─────────────────────┼────────────────────┼────────────────────────────────
  Core domain          │ Tier 1 amplio      │ Diferenciador competitivo —
  (Orders, Payments)   │                    │ máxima flexibilidad interna
  ─────────────────────┼────────────────────┼────────────────────────────────
  Supporting domain    │ Tier 2 estándar    │ Importante pero no crítico —
  (Notifications)      │                    │ equilibrio entre libertad y costo
  ─────────────────────┼────────────────────┼────────────────────────────────
  Generic domain       │ Mínima autonomía   │ Candidato a usar SaaS externo —
  (Auth, Email)        │                    │ no reinventar la rueda
```

**Pros de restringir autonomía en dominios genéricos:**
- Evita invertir tiempo en infraestructura que no diferencia al negocio
- Fuerza el uso de soluciones battle-tested (Auth0, SendGrid)

**Contras de restringir autonomía en dominios genéricos:**
- El equipo puede sentir que no tiene impacto real
- Las soluciones externas pueden no cubrir casos de uso específicos

### Factor 3: Requisitos de seguridad y compliance / Security and Compliance Requirements

```
  Requisito               │ Impacto en autonomía
  ────────────────────────┼────────────────────────────────────────────
  PCI-DSS (pagos)         │ Tier 3 estricto: logs, encryption, acceso
                          │ auditado — sin excepciones en Payment Service
  GDPR (datos personales) │ Tier 3: retención de datos, derecho al olvido
                          │ deben implementarse igual en todos los servicios
                          │ que manejen datos de usuarios
  SOC 2 / ISO 27001       │ Tier 3: controles de acceso, gestión de secretos,
                          │ patch management estandarizados
```

**Pros de forzar estándares de seguridad (Tier 3):**
- Superficie de ataque predecible y auditable
- Un solo parche de seguridad resuelve el problema en todos los servicios

**Contras:**
- Puede ralentizar a equipos que necesitan iterar rápido
- Los estándares pueden quedarse desactualizados si no se revisan periódicamente

### Factor 4: Escala organizacional / Organizational Scale

```
  Escala                  │ Estrategia recomendada
  ────────────────────────┼───────────────────────────────────────────────
  Startup (< 20 eng.)     │ Tier 3 mínimo, Tier 2 amplio
                          │ → velocidad > estandarización en etapa temprana
  Scale-up (20-100 eng.)  │ Formalizar Tier 3, definir Tier 2
                          │ → momento crítico: establecer estándares antes
                          │   de que la fragmentación sea irreversible
  Enterprise (> 100 eng.) │ Tier 3 robusto, Tier 2 bien definido
                          │ → el costo de la fragmentación supera al de
                          │   la burocracia de los estándares
```

> **ES:** La autonomía estructurada no es un estado fijo — evoluciona con la organización. Lo que es Tier 2 hoy puede convertirse en Tier 3 mañana si un problema recurrente justifica estandarizarlo, o puede subir a Tier 1 si la restricción demostró ser innecesaria.
>
> **EN:** Structured autonomy is not a fixed state — it evolves with the organization. What is Tier 2 today may become Tier 3 tomorrow if a recurring problem justifies standardizing it, or may move to Tier 1 if the restriction proved unnecessary.

### Factor 5: Tamaño e influencia del equipo DevOps / SRE / Size and Influence of the DevOps / SRE Team

**Español:**
El equipo de DevOps o SRE es quien opera, monitorea y mantiene la infraestructura de todos los servicios. Su capacidad real determina cuánta diversidad tecnológica puede soportar la organización — si son 3 personas, no pueden dominar 10 stacks distintos.

**English:**
The DevOps or SRE team is who operates, monitors, and maintains the infrastructure of all services. Their real capacity determines how much technological diversity the organization can support — if they are 3 people, they cannot master 10 different stacks.

```
Impacto del tamaño del equipo SRE / Impact of SRE team size:

  SRE pequeño (1-3 personas):
  → Tier 2 muy reducido — pocas opciones aprobadas
  → Cada nueva tecnología = carga operacional insostenible para el equipo
  → Autonomía de los equipos de desarrollo debe estar muy acotada
  → Regla práctica: si el SRE no lo puede operar a las 3am, no está aprobado

  SRE mediano (5-10 personas) con especialización:
  → Tier 2 moderado — pueden soportar 2-3 lenguajes, 3-4 DBs
  → Pueden tener sub-equipos: infra, observabilidad, seguridad
  → Autonomía guiada con catálogo de tecnologías aprobadas más amplio

  SRE grande o Platform Engineering team (> 10 personas):
  → Tier 2 amplio — pueden construir abstracciones (IDP: Internal Developer Platform)
  → Ofrecen "plataforma como producto": los equipos consumen servicios de la plataforma
  → Mayor autonomía de los equipos de desarrollo porque la plataforma absorbe complejidad
```

**Pros de un SRE grande con alta influencia:**
- Puede estandarizar la infraestructura y ofrecer más opciones aprobadas con bajo riesgo
- Los equipos de desarrollo se despreocupan de la operación y se enfocan en el dominio

**Contras de un SRE pequeño con poca influencia:**
- Si los equipos de desarrollo ignoran sus límites, el SRE queda desbordado
- La autonomía sin respaldo operacional genera deuda invisible que explota en producción

### Factor 6: Cultura de la empresa / Company's Culture

**Español:**
La cultura organizacional determina qué tan bien se aceptan y respetan los límites de autonomía. Una organización con cultura de confianza y responsabilidad puede darse el lujo de más autonomía; una con cultura de silos o política interna necesita más estándares explícitos.

**English:**
Organizational culture determines how well autonomy boundaries are accepted and respected. An organization with a culture of trust and accountability can afford more autonomy; one with a culture of silos or internal politics needs more explicit standards.

```
Cultura y su impacto en los límites / Culture and its impact on boundaries:

  Cultura de alta confianza / High-trust culture:
  → Los equipos respetan voluntariamente los estándares Tier 3
  → Los equipos proponen cambios a Tier 2 a través de procesos formales (RFC)
  → Autonomía en Tier 1 es real y no se cuestiona
  → Posible: más decisiones en Tier 1 y Tier 2

  Cultura de silos / Silo culture:
  → Cada equipo prioriza sus propios objetivos sobre los organizacionales
  → Los estándares Tier 3 se ignoran si no hay enforcement automático
  → Necesario: más restricciones formales, auditorías, CI/CD gates
  → Menos opciones en Tier 2, Tier 3 más extenso

  Cultura de experimentación / Experimentation culture:
  → Alta tolerancia al riesgo — los equipos quieren probar nuevas tecnologías
  → Necesario: proceso claro para añadir tecnologías a la lista aprobada de Tier 2
  → Sin ese proceso, la experimentación se convierte en fragmentación
  → "Innersource" / Architecture Decision Records (ADRs) como mecanismos de gobierno
```

**Pros de una cultura de alta confianza:**
- Los estándares se cumplen sin necesidad de enforcement costoso
- Los equipos proponen mejoras a los estándares → evolución continua del sistema

**Contras de una cultura de silos:**
- Los límites de autonomía deben ser técnicamente forzados (linters, pipelines, policies)
- El costo de governance aumenta proporcionalmente al tamaño de la organización
